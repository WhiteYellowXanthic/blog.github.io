---
layout: post
title:  "How to generate the distributed ID?"
date:   2019-07-15 00:00:00
categories: Java
---

Maybe you already know how to write a `snowflake` ID generator, and also you know combined it to `zookeeper` will help you to generate the IDs safely and rapidly, but I guess you do not wanna write it again when you need it.

Here is my code of this implementation.
```java
package com.github.zhangyanwei.common;

import com.github.zhangyanwei.common.exception.RuntimeException;
import org.apache.curator.framework.CuratorFramework;
import org.apache.zookeeper.KeeperException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Date;

import static com.google.common.base.Preconditions.checkState;
import static com.google.common.primitives.Longs.fromByteArray;
import static com.google.common.primitives.Longs.toByteArray;
import static java.lang.String.format;
import static org.apache.zookeeper.CreateMode.EPHEMERAL;

/**
 * 0 + TIME * 41 + SERVER * 10 + SEQ * 10
 */
public class IDGenerator {

    private static final Logger logger = LoggerFactory.getLogger(IDGenerator.class);

    private static final IDGenerator instance = new IDGenerator();
    private static final int MAX_10_BIT_VALUE = 0x3FF;

    private String zkRootPath;
    private String zkRunningNodePath;
    private int guardTime = 300000;
    private int guardTimeThreshold = (int) Math.ceil(guardTime * 0.2);
    private long baseMilliseconds;
    private int workingId;
    private int sequence;
    private long milliseconds;
    private volatile long maxMilliseconds;
    private boolean initialized;

    private IDGenerator() {}

    public synchronized static void initialize(CuratorFramework curatorFramework) {
        if (!instance.initialized) {
            instance.zkRootPath = "/id_services";
            instance.zkRunningNodePath = instance.zkRootPath + "/running";
            instance.guardTime = 300000;
            instance.guardTimeThreshold = (int) Math.ceil(instance.guardTime * 0.2);
            try {
                instance.baseMilliseconds = new SimpleDateFormat("yyyy-MM-dd").parse("2018-09-13").toInstant().toEpochMilli();
            } catch (ParseException ignore) {}
            instance.requestWorkingId(curatorFramework);
            instance.initialized = true;
        }
    }

    private void calculate() {
        long now = currentMilliseconds(baseMilliseconds);
        if (now > maxMilliseconds) {
            throw new RuntimeException("the max milliseconds not got updated," +
                    "maybe the zookeeper disconnected or other unknown reason.");
        }
        if (now == milliseconds) {
            if (++sequence > MAX_10_BIT_VALUE) {
                logger.warn("sequence overflow, waiting.");
                try {
                    wait(1);
                    calculate();
                } catch (InterruptedException ignore) {}
            }
        } else {
            milliseconds = now;
            sequence = 0;
        }
    }

    private synchronized long generate() {
        calculate();
        long result = 0;
        result = result | sequence;
        result = result | (workingId << 10);
        result = result | (milliseconds << 20);
        return result;
    }

    private void requestWorkingId(CuratorFramework curatorFramework) {
        try {
            boolean availableNodeExists = curatorFramework.checkExists().forPath(zkRunningNodePath) != null;
            int[] activeIds = availableNodeExists ?
                    curatorFramework.getChildren().forPath(zkRunningNodePath).stream()
                            .mapToInt(Integer::parseInt)
                            .sorted().toArray()
                    : new int[]{};

            for (int i = 0, offset = 0; i <= MAX_10_BIT_VALUE; i++) {
                if (activeIds.length == 0 || activeIds.length == offset || i != activeIds[offset]) {
                    try {
                        curatorFramework.create()
                                .creatingParentsIfNeeded()
                                .withMode(EPHEMERAL)
                                .forPath(zkRunningNodePath + "/" + i);
                        waitingGuardRelease(curatorFramework, i, 0);
                        startTimeGuardThread(curatorFramework, i);
                        workingId = i;
                        return;
                    } catch (KeeperException.NodeExistsException ignore) {}
                } else {
                    offset++;
                }
            }
        } catch (Exception e) {
            throw new RuntimeException("can't initialize the IDGenerator !", e);
        }

        throw new RuntimeException("working id exceeded !");
    }

    private void startTimeGuardThread(CuratorFramework curatorFramework, int i) throws Exception {
        Thread guardThread = new Thread(() -> {
            //noinspection InfiniteLoopStatement
            while (true) {
                try {
                    waitingGuardRelease(curatorFramework, i, guardTimeThreshold);
                } catch (Exception e) {
                    logger.error("can not update the guard time for ID generator, will try it 3 seconds later.", e);
                    try {
                        Thread.sleep(3000);
                    } catch (InterruptedException ignore) {}
                }
            }
        });
        guardThread.start();
    }

    private void waitingGuardRelease(CuratorFramework curatorFramework, int i, long threshold) throws Exception {
        String path = zkRootPath + "/" + i;
        boolean exists = curatorFramework.checkExists().forPath(path) != null;
        long currentTimeMillis = System.currentTimeMillis();
        long expireTime = exists ? fromByteArray(curatorFramework.getData().forPath(path)) : currentTimeMillis;
        long lifeTime = expireTime - currentTimeMillis;
        if (lifeTime > guardTime << 1) {
            throw new RuntimeException(format("please check the machine time, " +
                    "maybe it's too slower compare to the last ran time on given id: %s", i));
        }
        if (lifeTime > threshold) {
            Thread.sleep(lifeTime - threshold);
        }
        long nextExpireTime = System.currentTimeMillis() + guardTime + threshold;
        byte[] data = toByteArray(nextExpireTime);
        if (exists) {
            curatorFramework.setData().forPath(path, data);
        } else {
            curatorFramework.create().creatingParentsIfNeeded().forPath(path, data);
        }
        maxMilliseconds = nextExpireTime - baseMilliseconds;
    }

    public static long next() {
        checkState(instance.initialized, "IDGenerator not initialized yet !");
        return instance.generate();
    }

    private static long currentMilliseconds(long baseMilliseconds) {
        return new Date().toInstant().toEpochMilli() - baseMilliseconds;
    }

}
```