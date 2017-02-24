# prosemirror-firebase

Collaborative editing for ProseMirror using Firebase's Realtime Database. Features timestamped and attributed changes, version checkpointing, selection tracking, and more.

## usage

Assumes familiarity with [ProseMirror](http://prosemirror.net/guide/basics.html) and [Firebase](https://firebase.google.com/docs/database/web/start).

Call `new FirebaseEditor` with an object containing:
- `firebaseRef` [firebase.database.Reference](https://firebase.google.com/docs/reference/js/firebase.database.Reference)
- `stateConfig` [config](https://prosemirror.net/ref.html#state.EditorState^create) (must contain `schema`, `doc` will be overwritten)
- `view` Function that [creates an editor view](https://prosemirror.net/ref.html#view.EditorView.constructor) and returns it. Recieves an object containing:
    - `stateConfig` [config](https://prosemirror.net/ref.html#state.EditorState^create) to be used for creating the `state` for [`new Editor View`](https://prosemirror.net/ref.html#view.EditorView.constructor) (or [`new MenuBarEditorView`](https://github.com/ProseMirror/prosemirror-menu#class-menubareditorview))
    - `updateCollab` Function that should be called in [`dispatchTransaction`](https://prosemirror.net/ref.html#view.EditorProps.dispatchTransaction) with the `transaction` and the `newState`:

        ```js
        dispatchTransaction(transaction) {
            let newState = view.state.apply(transaction)
            view.updateState(newState)
            updateCollab(transaction, newState)
        },
        ```

    - `selections` Object with keys being `clientId`s and values being [selections](https://prosemirror.net/ref.html#state.Selection) (useful for [showing cursors and selections of other clients](#cursors-and-selections))
- `clientId` (optional, used to attribute changes and distinguish them from those of other clients, defaults to a [unique 120-bit string identifier](https://gist.github.com/mikelehen/3596a30bd69384624c11))

Returns a [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) that resolves when the editor is ready (when existing content has been loaded and an editor view has been created). The resolved value is an object containing the properties and methods of the editor:
- `view` (see above)
- `selections` (see above)
- `destroy` Function that removes all database listeners that were added, removes the client's selection from the database, and calls [`editorView.destroy`](https://prosemirror.net/ref.html#view.EditorView.destroy)

### basic

```js
new FirebaseEditor({
    firebaseRef: /*database.Reference*/,
    stateConfig: {
        schema: /*Schema*/,
    },
    view({ stateConfig, updateCollab }) {
        let view = new EditorView(/*dom.Node*/, {
            state: EditorState.create(stateConfig),
            dispatchTransaction(transaction) {
                let newState = view.state.apply(transaction)
                view.updateState(newState)
                updateCollab(transaction, newState)
            },
        })
        return view
    },
})
```

### cursors and selections

```js
function stringToColor(string, alpha = 1) {
    let hue = string.split('').reduce((sum, char) => sum + char.charCodeAt(0), 0) % 360
    return `hsla(${hue}, 100%, 50%, ${alpha})`
}

new FirebaseEditor({
    /*...*/
    view({ stateConfig, updateCollab, selections }) {
        let view = new EditorView(/*dom.Node*/, {
            /*...*/
            decorations({ doc }) {
                return DecorationSet.create(doc, Object.entries(selections).map(
                    function ([ clientID, { from, to } ]) {
                        if (from === to) {
                            let elem = document.createElement('span')
                            elem.style.borderLeft = `1px solid ${stringToColor(clientID)}`
                            return Decoration.widget(from, elem)
                        } else {
                            return Decoration.inline(from, to, {
                                style: `background-color: ${stringToColor(clientID, 0.2)};`,
                            })
                        }
                    }
                ))                  
            },
        })
        return view
    },
})
```

### ready

```js
new FirebaseEditor({
    /*...*/
}).then(({ view, selections, destroy }) => {
    console.log('editor ready')
}).catch(console.error)
```

## license

[MIT license](LICENSE).
