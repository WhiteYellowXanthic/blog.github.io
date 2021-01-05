---
layout: post
title:  "Create a custom form editor"
date:   2019-06-04 08:00:00
categories: Javascript
---

These days, I've investigated how to provide a convenient way for the maintainers to design/edit the form.

In our company, already developed an OA system, which allows the administrators to design the forms online, but the current implementation is to edit the `HTML` and `Javascript` directly, that makes the administrators irritable.

To make this job easy and efficient, need to allow them to design and edit the forms in a `WYSIWYG` editor, and bind the actions to the input components directly(it's transparent to the administrators).

I've chosen `GrapesJS` after I compared multiple ways to do that.

|-------------|--------|
|**InfoPath**|It's familiar to administrators, but need to reedit the generated HTML to make them dynamic. Additional, it's not a standalone application. | 
|**LibreOffice**|Can use it to design the form, but lack of documentation, and hard to extend.|
|**WYSIWYG HTML Editor**|Most of this kind of editors are aims to write the documentation. They're not for form design.|

The documentation of `GrapesJS` is enough for basic knowledge, for deep customization, need to read the source code (they are well-organized.)

Here is a sample for how to write a `GrapesJS` plugin, and how to add the custom toolbar items.
```javascript
function tableEditorPlugin(editor) {
  const Command = editor.Commands.get('select-comp')
  editor.Commands.extend('select-comp', {
    // Overwrite this private method to make the toolbar conditionally display.
    updateToolbar(mod) {
      const em = this.config.em
      const model = mod == em ? em.getSelected() : mod
      if (model) {
        const toolbar = model.get('__toolbar') || model.get('toolbar')
        const filtered = toolbar.filter(item => item.isAvailable === undefined || item.isAvailable.call(this, em))
        model.has('__toolbar') || model.set('__toolbar', toolbar)
        model.set('toolbar', filtered)
      }
      Command.updateToolbar.call(this, mod)
    }
  })

  // Command: Append a row
  editor.Commands.add('table:table-row-add', editor => {
    const em = editor.getModel()
    const component = editor.getSelected()
    const parent = component.parent()
    parent.append({
      type: 'row',
      components: Array(component.components().length).fill({ type: 'cell' })
    })
    component.emitUpdate();
  })

  // Command: Combine the cells
  editor.Commands.add('table:table-cells-combine', editor => {
    const em = editor.getModel()
    const components = editor.getSelectedAll()
    // Keep the first selected component, but remove others.
    const component = components[0]
    const parent = component.parent()

    function update(rowIndex, columnIndex, colspan, ...components) {
      const targetParent = parent.collection.at(rowIndex)
      const columns = targetParent.components().toArray()
      columns.splice(columnIndex, colspan, ...components)
      targetParent.components(columns)
    }

    // Update the attributes
    const {rowspan, colspan, tl} = component.get('__span') || {}
    const [rowIndex, columnIndex] = tl
    // Clear the select status because it will cause errors.
    em.setSelected(null)
    for (let i = rowIndex + 1, end = rowIndex + rowspan; i < end; i++) {
      update(i, columnIndex, colspan)
    }
    component.setAttributes({rowspan, colspan})
    update(rowIndex, columnIndex, colspan, component)
    // Reset the selection status
    em.setSelected(component)
  })

  // Command: Split the cell
  editor.Commands.add('table:table-cell-split', editor => {
    const em = editor.getModel()
    const component = editor.getSelected()
    const parent = component.parent()

    const columnIndex = component.index()
    const rowIndex = parent.index()
    const shadowAttributes = {...component.attributes.attributes}
    const {colspan, rowspan} = shadowAttributes

    function update(rowIndex, columnIndex, colspan, skip = 0) {
      const targetParent = parent.collection.at(rowIndex)
      const columns = targetParent.components().toArray()
      columns.splice(columnIndex + skip, 0, ...Array(colspan - skip).fill({ type: 'cell' }))
      targetParent.components(columns)
    }

    component.setAttributes({...shadowAttributes, colspan: undefined, rowspan: undefined})
    update(rowIndex, columnIndex, colspan, 1)
    for (let i = rowIndex + 1, end = rowIndex + rowspan; i < end; i++) {
      update(i, columnIndex, colspan)
    }
  })

  // Extend the row toolbar items.
  editor.DomComponents.addType('row', {
    extendFn: ['initToolbar'],
    model: {
      initToolbar() {
        const model = this
        var tb = [...model.get('toolbar') || []]

        // Tool: append a row
        tb.push({
          attributes: { class: 'fa fa-plus' },
          command: ed => ed.runCommand('table:table-row-add')
        })
        model.set('toolbar', tb)
      }
    }
  })

  // Extend the cell toolbar items.
  editor.DomComponents.addType('cell', {
    extendFn: ['initToolbar'],
    model: {
      initToolbar() {
        const model = this
        var tb = [...model.get('toolbar') || []]

        // Tool: combine the cells.
        tb.push({
          attributes: { class: 'fa fa-square-o' },
          command: ed => ed.runCommand('table:table-cells-combine'),
          isAvailable (ed) {
            const components = ed.getSelectedAll()
            components.forEach(component => component.unset('__span'))
            if (components.length > 1) {
              const [ tl, br ] = components
                .reduce((r, comp) => {
                  const [ tl, br ] = r
                  const parent = comp.parent()
                  if (parent) {
                    // The parent may be undefined, ex: the component was removed before the undo command(Ctrl + Z).
                    const rowIndex = parent.index()
                    const columnIndex = comp.index()
                    tl[0] < rowIndex || (tl[0] = rowIndex)
                    tl[1] < columnIndex || (tl[1] = columnIndex)
                    br[0] > rowIndex || (br[0] = rowIndex)
                    br[1] > columnIndex || (br[1] = columnIndex)
                  }
                  return r
                }, [[], []])
              const rowspan = br[0] - tl[0] + 1
              const colspan = br[1] - tl[1] + 1
              const selected = components[0]
              selected.set('__span', {rowspan, colspan, tl, br})
              return components.length === colspan * rowspan
            }
            return false
          }
        })
        tb.push({
          attributes: { class: 'fa fa-columns' },
          command: ed => ed.runCommand('table:table-cell-split'),
          isAvailable (ed) {
            const components = ed.getSelectedAll()
            if (components.length === 1) {
              const attributes = components[0].attributes.attributes
              return attributes.colspan || attributes.rowspan
            }
            return false
          }
        })
        model.set('toolbar', tb)
      }
    }
  })

  // Block: Header
  editor.BlockManager.add('header', {
    label: '表单标题',
    attributes: { class:'gjs-fonts gjs-f-button' },
    content: `<h1></h1>`
  })

  // Block: Form
  editor.BlockManager.add('form', {
    label: '表单',
    attributes: { class:'gjs-fonts gjs-f-hero' },
    content: `<form>
      <table style="width: 100%">
        <thead>
          <th><h1>Title</h1></th>
        </thead>
        <tbody>
          <tr>
            <td></td>
          </tr>
        </tbody>
      </table>
    </form>`
  })

  // Block: Div
  editor.BlockManager.add('div', {
    label: '容器',
    attributes: { class:'gjs-fonts gjs-f-b1' },
    content: `<div><span>content</span></div>`
  })

  // Block: Text
  editor.BlockManager.add('text', {
    label: '输入框',
    attributes: { class:'gjs-fonts gjs-f-text' },
    content: `<input type="text">`
  })

  // Block: Textare
  editor.BlockManager.add('textarea', {
    label: '多行文本输入框',
    attributes: { class:'gjs-fonts gjs-f-quo' },
    content: `<textarea></textarea>`
  })
}
```
