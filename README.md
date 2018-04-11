# Optic Editor Plugin Spec v1
Specification for Optic compatible text editor plugins

# tl;dr
1. Open a websocket connection 
2. Listen for 1 event
3. Send 2 events from onChange callbacks. 
4. Publish Your Work, Become an Optic Contributer 
5. Swag Arrives in Mail

## Existing Plugins 
* [Atom](https://github.com/opticdev/optic-atom-plugin) - maintained by the Optic Team

## Contributing a Plugin 
Any individual or team is invited to contribute an editor plugin to Optic's ecosystem. Optic's team will provide support during the development process and can be available to help anytime. If you want your plugin to become the official one listed on our website email Aidan (aidan@opticdev.com). A preference will be given to teams that intend to maintain their plugin. Plugins will be evaluated and tested against this spec before being listed. 

Individuals will be listed prominently as Optic contributors and their organizations (optionally) as sponsors. Swag will be mailed :)

## Overview 
Optic Editor plugins forward the current state of an IDE to Optic's local server `localhost:30333` over a websocket connection. The state includes cursor position, highlighted region and any changes made in memory. Optic uses this information to establish the user's context and query the appropriate resources. The results of this processing are not handled by the editor plugins and are instead forwarded to the Optic GUI. 

![alt text](https://github.com/opticdev/docs/raw/master/images/system-overview.svg?sanitize=true)

# Spec
**Terms:**
* Server - the local Optic server
* Editor - the target text editor or IDE you are building for. 
* Plugin - your plugin for Editor

## Connecting to Optic
Users may start Optic after the editor, at the same time as the editor or even hours before. For this reason we can't assume that the server will even be running when our plugin initializes. Your plugin should connect to the server if it is active or continue trying to connect at a regular interval of no more than 10 seconds. These pings are pretty inexpensive especially when spread out over 10 seconds. 

Each editor should connect to Optic with its own distinct path: `localhost:30333/socket/editor/:editorName`. Editor name should be lowercase and whitespace-free. ie Atom -> atom, Visual Studio -> visualstudio

Test Cases: 
> 1. Editor Started. Wait 10 seconds. Start Optic. Within 10 seconds editor should connect to Optic. 
> 2. Editor is connected to Optic. Optic is turned off. Wait 10 seconds. Start Optic. Within 10 seconds editor should connect to Optic. 

## Sending & Receiving Events
If your editor supports a node runtime for plugins, you're in luck! Optic has released the [optic-editor-sdk](https://github.com/opticdev/optic-editor-sdk/blob/master/src/EditorConnection.js). The Editor SDK automatically manages creating your socket connection, sending & receiving events and reconnecting if the connection is lost. It'll cover everything but the editor specific events and logic. 

If you aren't using a node environment you'll have to implement the protocols covered below manually. Don't be deterred, it's still pretty simple. 

## Monitoring 
Plugins should monitor for two kinds of editor events. Each editor has a different name for these events so we'll define them here in the abstract. 

1. **cursor position moved** - Anytime a user moves their cursor (either by clicking, arrow keys or other shortcut). This also includes selecting a region of text with the mouse.

2. **text content changed** - Anytime a change is made by using the editor. Some editors are constantly checking the disk for external changes from tools like Git, others don't notice these kinds of changes unless the user explicitly reloads the file. We want whatever the editor is displaying to be what Optic is reading (otherwise there's a mismatch that'll confuse the user). Follow the convention of the editor with your plugin. Most of the time this won't be an issue if you listen for the editor's updated event. 

After each of these events you should send the following JSON message through the websocket: 
```json
{"event": "context", "file": "full/path/to/file", "start": 133, "end": 133, "content": "entire contents of file"}
```
A note on start/end: If your editor stores ranges as column/row you will have to convert them to absolute indices. Optic expects the first character in a file to be 0, the second 1, etc. 

Plugins should assume that Optic is stateless and send the full event every time. We tinkered around with sending a buffer of content changes, but found it was more trouble and much more error prone than it was worth. All the IO is local so optimizing for smaller payloads isn't as important as it would be in a non-local networked environment. 

Test Cases: 
> 1. Editor is connected to server. Click somewhere in a file, Optic event sent with the new range to server
> 2. Editor is connected to server. Use arrow keys to navigate around a file, Optic event sent with the new range to server
> 3. Editor is connected to server. Select text by dragging a range, Optic event sent with the new range to server
> 4. Editor is connected to Optic. Text is \[added, deleted\], Optic event sent with new range and content to server

## Capturing Search 
One of the cool features of Optic is the ability to search just by typing in your IDE. Today we use '///' to let Optic know a search is being made. It's on our roadmap to stop using leading characters to indicate a search, but right now we don't have a good alternative implemented. 

Editor plugins are therefore responsible for figuring out if they should send a `context` event or a `search` event. You should add your logic for capturing searches to your **text content changed** callback. If the user is typing a search opt to send a search event to Optic and in all other cases send a context event. 

```json
{"event": "search", "file": "full/path/to/file", "start": 133, "end": 133, "content": "entire contents of file", "query": "User's query"}
```

### Editor SDK
If you're in a node environment the Editor SDK ships with logic for this. Just pass in the content of the current line and start/end positions of the cursor within that line. 
```javascript
import { checkForSearch } from 'optic-editor-sdk'
checkForSearch(currentLineContent, startInLine, endInLine)
///query                                                       {isMatch: true, query: query}
/// hello                                                      {isMatch: true, query: hello}
//normal comment                                               {isMatch: false}
class Test {                                                   {isMatch: false}
```

### Other Implimentations 
If you aren't in a node environment you'll have to implement the search check yourself. Here's our regex & node code as a reference:
```javascript
const searchRegex = /^[\s]*\/\/\/[\s]*(.+)/
export function checkForSearch(line, start, end) {

	const match = searchRegex.exec(line)

	const isSearch = (start === end) && match !== null
	return {
		isSearch,
		query: (match !== null) ? match[1].trim() : undefined
	}
}
```

Test Cases: 
> 1. Editor is connected to server. Type '///query', Optic search event sent, no context event sent. 
> 2. Editor is connected to server. Type '//query', Optic context event sent, no search event event sent.
> 3. Editor is connected to server. Write normal code, Optic context event sent, no search event event sent.

## File Staging Events
Many editors will not read changes to disk made by outside processes. Early versions of Optic would save their updates to the disk but the editors wouldn't display the updates. This necessitated an event being sent from the server to the plugin that would stage new contents in the editors. The file updates are also saved to disk by the server. 

Your plugin should listen for an event called `files-updated`. Within that payload will be an object called `updates` which has  the absolute path of each file that needs updating as its keys and the new contents as the values. You should iterate over all of the files and set their content in the editor to the new value. Also tell the editor to open any files that were changed but are not currently open. 

```javascript
{
	"event": "files-updated",
	"updates": {
		"/path/to/file1": "new contents as string",
    "/path/to/file2": "new contents as string"
	}
}
```

**Some gotchas we've run into**: 
* Some editors with complex tabbing systems will require you to set the text of EVERY tab that is open for the file path. 
* Some editors will not like having text updated in this manner if there are pending/unsaved changed. Make sure you apply any `force` flags that are provided. 
* Don't refresh the tabs instead of setting their content. Optic saves everything asynchronously and while it only takes a few milliseconds to hit the disk we have observed some race conditions when we tried that approach. 

Test Cases: 
> 1. Optic is connected to server. Receives an update event with 3 file changes. All tabs update their contents
> 2. Optic is connected to server. Receives an update event with 1 file (that is closed). That file opens in a new tab with updated content


# Testing
Testing plugins is easiest if you capture the server output. All connections from editors will get logged as will all received and sent events. You can capture the console output by opening the Optic App from the command line. 

Run Optic.app from a terminal session: 
```bash
aidancunniffe$ /Applications/Optic.app/Contents/MacOS/Optic
```
You should see data like this come through live. Note that many of the events you'll see pop up are from are being sent to/from the Optic GUI and are not going to be routed to your plugin. 
```
STARTING SERVER
Killed last server pid:14431
Server online at http://localhost:30333/
Press RETURN to stop...

STARTING CHIP NOW
READY
Finished navigating to url Optional(file:///Applications/Optic.app/Contents/Resources/embedded/webapp/dist/index.html#/)
optic-agent agent connected

atom editor connected

RECEIVED {"event":"context","file":"/Users/aidancunniffe/Desktop/demo/demo_project/models.js","start":253,"end":253,"contents":"const mongoose = require('mongoose')\n\nconst model = mongoose.model('Hello', new mongoose.Schema({\n    'isAdmin': 'boolean',\n    'firstName': 'string',\n    'test': 'string',\n    'lastName': 'string',\n}))\n\napp.post('/hello', function (req, res) {\n  new Model({ isAdmin: req.body.isAdmin,\n  firstName: req.body.firstName,\n  test: req.body.test,\n  lastName: req.body.lastName }).save((err, item) => {\n    if (!err) {\n        res.send(200, item)\n    } else {\n        res.send(400, err)\n    }\n  })\n})\n\napp.post('/hello', function (req, res) {\n  new Model({ isAdmin: req.body.isAdmin,\n  firstName: req.body.firstName,\n  lastName: req.body.lastName }).save((err, item) => {\n    if (!err) {\n        res.send(200, item)\n    } else {\n        res.send(400, err)\n    }\n  })\n})\n"}

```
