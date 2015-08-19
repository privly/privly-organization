# Seamless-Posting Architecture Documentation

## TL;DR

This documentation is supposed to explain the architecture and the implementation detail of the seamless-posting feature of the Privly extension.

## What's Seamless-Posting

Seamless posting is a method that puts the extension's posting form element directly over the form element of the host page. It is 'Seamless' because the user does not need to leave a web app in order to post encrypted content to it.

## What is the work flow of Seamless-Posting

In seamless-posting, we first create a new Privly link and place it in the host page's form element. While the user types content into the protected form element, the content associated with the link continually updates. If the user clicks the cancel button, the content associated with the Privly link will be destroyed and the link will be removed from the host page's form element.

![img-1](https://raw.githubusercontent.com/privly/privly-organization/master/graphics/diagrams/seamless-posting/img-1.png)

## What are the roles of Privly-App and Privly-Extension?

In general, content scripts in privly-extension (like privly-chrome) are in charge of providing user interface to start seamless-posting and inserting the iframe of privly-application, while the privly-application inside the iframe is in charge of creating links (or loading links), updating links and destroying links depends on user action.

![img-2](https://raw.githubusercontent.com/privly/privly-organization/master/graphics/diagrams/seamless-posting/img-2.png)

### Privly-Application

Seamless-posting implements the following parts:

1. Seamless-posting template (called prototype view) and related JavaScript (called view adapter)
   
   > prototype view: `privly-application/templates/seamless.html.template`
   > 
   > view adapter: `privly-application/shared/javascripts/viewAdapters/seamless.js`
  
   This is the view layer. It will be loaded into an iframe (to ensure safety) and put inside the host page. Users will write secure messages in this iframe that sits directly over the form of the host page.

   The default template provides a textarea with a green background color. The view adapter calls the network interfaces to update the link as the user types into the text area.
   
   This is a "prototype view" because it is a base template that can be extended/inherited in the specific Privly Application.

2. Seamless-posting TTLSelect view and view adapter

   > prototype view: `privly-application/templates/seamless_ttlselect.html.template`
   > 
   > view adapter: `privly-application/shared/javascripts/viewAdapters/seamless_ttlselect.js`

   This view layer provides menu-style UI for user to change the seconds_until_burn(TTL) option. This will be loaded into an iframe above the seamless-posting form if the user hovers over the Privly button.

3. Seamless-posting feature for Message and PlainPost application

   Privly-Apps implement the seamless-posting and seamless-posting TTLSelect views.
   
   > Message App:
   > 
   > view: `privly-application/Message/seamless.html.subtemplate`
   > 
   > controller: `privly-application/Message/js/controller/seamless.js`
   > 
   > PlainPost App:
   > 
   > view: `privly-application/PlainPost/seamless.html.subtemplate`
   > 
   > controller: `privly-application/PlainPost/js/controller/seamless.js`

   See `privly-application/Message/js/controllers/seamless.js` for samples of hooking an app into the seamless form.

   See `privly-application/Message/js/messageModel.js` for samples of manipulating Privly links.

### Privly-Extension (Privly-Chrome)

To support seamless-posting, the extension includes the following content scripts:

1. Seamless-posting content script
   
   > Content script:
   > 
   > `javascripts/content_scripts/posting.*.js`
   > 
   > The content script is split into several files for better readability.
   
   The content scripts:
   - Create a Privly button at the top-right corner when user focus on an editable element in the host page (`posting.button.js`).
   - Provides tooltip (`Clicks to enable Privly posting`) when user hovers on the button (`posting.tooltip.js`)
   - Creates the iframe of privly-application seamless-posting view (`app`) when user clicks the button (`posting.app.js`)
   - Inserts Privly link into the original editable element (`target`) after the view in iframe creates a Privly link (`posting.target.js`)
   - Provides a dropdown menu (`TTLSelect`) when user hovers on the button after enabling seamless-posting (`posting.ttlselect.js`)
   - Destroys the iframe if user clicks the button again (`posting.app.js`)
   
   Notice that most of the features above involve more than one content script file. See implementation section below for details.

2. Seamless-posting background script
   
   > Background script:
   > 
   > `javascripts/background_scripts/posting_process.js`
   > 
   > `javascripts/background_scripts/context_menu.js`
   > 
   > `javascripts/background_scripts/modal_button.js`
   
   The background script:
   - Creates a bridge for communicating between the content script and the Privly application (for example, which link to insert) (`posting_process.js`)
   - Pops up login dialog if the user is not logged in (`posting_process.js`)
   - Creates context menu for the user to select the desired App and enable seamless-posting (`context_menu.js`)
   - Update the icon of the action_button (`modal_button`) according to the scripting context that can capture keyboard inputs (`modal_button.js`)

## Integrating Seamless-Posting with Privly-Applications

> ECMAScript 6 Promise is heavily used to arrange the asynchronous callback order.

### Seamless-Posting View Adapter

#### Initialize process (`start`):

The real workflow of seamless-posting is a bit different from the flow diagram above because seamless-posting also take original content of the editable element into account. Take GitHub as an example, when you are posting a comment using seamless-posting, you are posting a comment containing a Privly link in fact. When you want to edit the comment, you will get a textarea contains that Privly link instead of your protected content. How to edit the protected content inside the link? In the flow diagram above, our posting process is seamless, but our editing process is not seamless.

In short, if we start seamless-posting in such *editing* circumstance (the content of the editable element already contains a Privly link when we start seamless-posting), we are expecting to use our real comment as the initial content, not empty content, nor that Privly link.

The detailed flow diagram of the initializing process of seamless-posting:

![img-3](https://raw.githubusercontent.com/privly/privly-organization/master/graphics/diagrams/seamless-posting/img-3.png)

1. *(Privly-Chrome appends the iframe into the host page.)*

2. Send message to content script to switch the icon of Privly button to spinner icon (`msgStartLoading`).

3. Check connection.

  1. If fails: send message to content script to destroy the iframe (`msgAppClosed`) and restore the Privly button icon to lock icon and stops (`msgStopLoading`) and pops up login dialog (`msgPopupLoginDialog`).

  2. If succeeds: goto 4.

4. Send message to content script to retrive the original content of the editable element (`target`) (`msgGetTargetText`).

5. Check whether the original content contains a Privly link.

  1. If contains: try to load it (`loadLink`).

    1. If loading succeeded and the link is a valid Privly link and the user has edit permission: Use the content of the Privly link as the initial content of the posting form (`initial content = (content of target).replace(the privly link, the content of the privly link)`), Goto 9.

    2. Else: Goto 6.

  2. If not contains: Goto 6.

6. Create a new and empty Privly link (`createLink`) and use it as the main link.

7. Send message to content script to insert the link into the editable element (`target`) (`msgInsertLink`).

8. Initialize using a new link completes. Goto 10.

9. Initialize using an existing link completes. Goto 10.

10. Set up targetContentMonitor (`beginContentClearObserver`), monitoring whether the content of the target contains our main link, once cleared, send message to content script to destroy this iframe (`msgAppClosed`).

11. Send message to content script switch the icon of Privly button from spinner icon to original icon (`msgStopLoading`).

12. Send message to content script to notify the completion of initial process (`msgAppStarted`).


#### Destroy process:

*(Privly-Chrome tries to destroy the iframe)*

2. *(Privly-Chrome send message to the iframe that it is going to be destroyed)*

3. Destroy the main link (`deleteLink`).

4. Send message to content script to notify the closing (`msgAppClosed`).

5. *(Privly-Chrome removes iframe from DOM tree)*

#### Create link (`createLink`):

After creating a new Privly link, view adapter will call `application.postpcoessLink` if it is implemented. This allows Privly applications to manipulate the link. For example, the Message app will append the encryption key into the link.

1. Call `privlyNetworkService.sameOriginPostRequest` to create a link.

2. Call Application Model to post-process the link (`application.postprocessLink`).

3. Emit `afterCreateLink` event for Application Models.

4. Use the link as the main link.

#### Update link (`updateLink`):

When updating a link, obviously we need to cancel the last updating ajax request if it is still in progress. This function will be called when user inputs in the textarea of the seamless-posting form of the Privly-application (see "When user input something" section below).

1. Cancel last ongoing update request.

2. Call Application Model to transform textarea content into structured content (`application.getRequestContent`).

3. Call `privlyNetworkService.sameOriginPutRequest` to update the main link according to the structured content.

4. Emit `afterUpdateLink` event for Application Models.

#### Delete link (`deleteLink`):

The `deleteLink` function will be called in the destroy process.

1. Send message to content script to switch the icon of Privly button to spinner icon (`msgStartLoading`).

2. Call `privlyNetworkService.sameOriginDeleteRequest` to delete the main link

3. Emit `afterDeleteLink` event for Application Models.

4. Send message to content script to switch the icon of Privly button to original icon (`msgStopLoading`).

#### When user inputs something (`onHitKey`):

When user inputs something, the Privly link should get updated according to the content and the `seconds_until_burn` options. Besides, for circumstance like Facebook Chat, there is no submitting buttons: the form is submitted when user presses enter key. Thus when user presses the enter key in our textarea of the seamless-posting form, we need to emulate the enter key event in the editable element. This behaviour is called keyboard forwarding.

When forwarding the enter key event, we also forward modify keys (like shift key, control key, alt key, etc). This allows user to use submitting shortcut keys provided by the host page (if available) even in the seamless-posting form.

Notice that, we have set the `targetContentMonitor` in the initialize process (see step 10 of the "Initialize process" section), thus when the content of editable element is cleared (which is a normal behaviour when the form is being submitted), our seamless-posting form will be automatically closed.

1. Update link (`updateLink`)

2. If the pressed key is `enter`: send message to content script to simulate the `enter` key on the editable element (`target`) (`msgEmitEnterEvent`)

> Sending message to content script is achieved by sending message to background script and background script forwarding message to the content script.

### Seamless-Posting TTLSelect View Adapter

#### Initialize process (`start`):

The complete process of initializing a TTLSelect iframe is complicated. It involves with many communications between content script and Privly app due to the following reasons:

- The TTLSelect appears above the Privly button, if there is enough space. Otherwise it appears below the Privly button.

- The options of the TTLSelect is unknown and it should be retrived from the Privly application model.

- For the above reason, the size of the TTLSelect iframe is unknown before initializing.

- We want the first item of the TTLSelect always appears closest to the user mouse cursor and the last item always appears farest to the user mouse cursor. The order of the TTLSelect items are determined after its position is determined.

Thus, to initialize a TTLSelect, we have the following steps:

1. Get TTL options from Application Model (`application.getTTLOptions`)

2. Calculate width and height
   
   > Notice: DOMs of the select items are not created at this moment. Only positions are calculated.

3. Send message to content script to notify the completion of loading, containing the width and height

4. *(Privly-Chrome resize the iframe and calculates the position of the iframe, based on the width and height. The iframe may be below the button, or above the button)*

5. *(Privly-Chrome send message to iframe to notify whether it is below the button or above the button)*

6. Generate menu DOM according to position: smaller options are always closer to the mouse-pointer.

7. *(Privly-Chrome fade in and show the iframe)*

#### When user clicks something (`onItemSelected`):

Keep in mind that the TTLSelect App is inside a different iframe compared to the seamless-posting form:

```
Host page
  - iframe
    - Seamless-posting form app
  - iframe
    - Seamless-posting TTLSelect app
```

To send a message from TTLSelect to posting form, we use content script as a bridge, instead of directly send messages using background script as a bridge:

```
+-----------+          +--------------+          +-----------+
| TTLSelect | -> BG -> |Content Script| -> BG -> | Post Form |
+-----------+          +--------------|          +-----------+
```

The step is simple:

1. Send message to content script to notify that user has clicked an option (`msgTTLChange`)

> Sending message to content script is achieved by sending a message to the background script and background script forwarding message to the content script.

## Implementation of Privly-Chrome

### Content Script

#### Resource, ResourceItem

We have to manage state for editable elements on the page (for example, whether it is in seamless-posting mode). We also need to manage state for programmatically-created elements (for example, a Privly button may be spinner icon, or lock button, or cancel button, based on state). Besides, we also need to manage some global objects (for example, a Privly button may contain a timer to postpone the hiding process). In addition, those stuff should be treated together: when editable element is removed, our state data should be cleared, our Privly button DOM related to that editable element should be removed and our Privly button timer should be canceled.

Thus we created the `Resource` class (implemented in `posting.resource.js`), to provide a container for those components (`ResourceItem`, implemented in `posting.resource.js`), for a specific editable element.

Features:

- Different editable elements are linked to different `Resource`s.

- A `Resource` can hold many different `ResourceItem`s.

- A Privly button DOM is managed by a `ResourceItem` (implemented in `posting.button.js`), an editable element DOM itself is managed by a `ResourceItem` (implemented in `posting.target.js`), a Privly button tooltip DOM is managed by a `ResourceItem` (implemented in `posting.tooltip.js`), a Privly seamless-posting Application iframe DOM is managed by a `ResourceItem` (implemented in `posting.app.js`), etc.

  ![img-4](https://raw.githubusercontent.com/privly/privly-organization/master/graphics/diagrams/seamless-posting/img-4.png)

- There is a global pool of `Resource`s (managing all `Resource` instance).

- `ResourceItem` has a `destroy` method, which is a kind of destructor.

- Different `ResourceItem` can have different destructor behaviors, for example, for a Privly button Resource Item, when it is destroyed, the DOM should be removed from DOM tree. However for a Target Resource Item, when it is destroyed, the DOM (the editable element itself) should not be removed from DOM tree. Notice that, Privly button Resource Item also cancels its timers inside `destroy`.

- Some `ResourceItem` may not contain DOM nodes, for example, `posting.controller.js` implements a `ResourceItem` which only control other `ResourceItem`s.

- `ResourceItem`s inside the same `Resource` can communicate with others by calling `broadcastInternal` (It is the observer design pattern).

- Each `Resource` has a unique id, theoretically (even across all tabs).

- When a Privly Application want to communicate with a specific `Resource`, it need to provide the id of the `Resource`.

- The id of the `Resource` is passed by URI querystring to the privly-application when iframe is created, thus the application can send message back later.

- When `Resource` receives a message, it will forward it to its `ResourceItem`s.

- A message received by `Resource` can be external -- Chrome message from background script, or internal -- by calling `broadcastInternal()`.

  ![img-5](https://raw.githubusercontent.com/privly/privly-organization/master/graphics/diagrams/seamless-posting/img-5.png)

- Messages sent from `broadcastInternal` do not support `sendResponse`.

- A `ResourceItem` can subscribe different kind of message (differentiate by `action` property) by calling `addMessageListener`.

- We appoint that, messages of which `action` start with `posting/internal/` are internal messages (which means, when you are using `broadcastInternal`, the `action` property of your messages should be `posting/internal/` prefixed).

- The background script will forward Chrome messages of which `action` start with `posting/app` to all Privly applications (each Privly applications will filter messages).

- The background script will forward Chrome messages of which `action` start with `posting/contentScript` to all content scripts (each `Resource` will filter messages as mentioned above).

  ![img-6](https://raw.githubusercontent.com/privly/privly-organization/master/graphics/diagrams/seamless-posting/img-6.png)

- The background script itself will try to handle Chrome messages of which `action` start with `posting/background`.

#### Resource State

A `Resource` has a `state` property that indicates whether it is in the seamless-posting mode.

`state == OPEN`: In seamless-posting mode (posting form is open).

`state == CLOSE`: Not in seamless-posting mode (posting form is closed).

#### Button Internal State

Button has three kind of icons, `lock`, `spinner` and `cancel`.

Button also has two properties indicates its state:

`loading`: whether the button should show a spinner icon.

`state`: the same to Resource State.

We separated the two properties thus each `ResourceItem` can set one of them without caring about affecting real states.

- For `loading == true`: internal state is `LOADING`, The button will show `spinner` icon.

- For `loading == false` and `state == CLOSE`: internal state is `CLOSE`, the button will show `lock` button.

- For `loading == false` and `state == OPEN`: internal state is `OPEN`, the button will show `cancel` button.

`INTERNAL_STATE_PROPERTY` defines behaviors of the button in each internal state:

```js
var INTERNAL_STATE_PROPERTY = {
  CLOSE: {
    autohide: true,  // whether the button should hide after seconds
    clickable: true, // whether the button is clickable
    tooltip: true,   // whether to show the tooltip when hovering on the button
    icon: SVG_OPEN   // the SVG of the icon when button is in this state
  },
  OPEN: {
    autohide: false,
    clickable: true,
    tooltip: false,
    icon: SVG_CLOSE
  },
  LOADING: {
    autohide: false,
    clickable: false,
    tooltip: false,
    icon: SVG_LOADING
  }
};
```

#### ContextId, ResId, AppId

Each iframe in each tab contains a unique `contextid`, which is generated in `context_messenger.js`, to filter contexts for messages (notice that every Chrome message is broadcasted to all tabs, so we filter the message at the context layer first).

Each `Resource` contains a unique `resid`, which is used to tell Privly applications that where to send Chrome message back later. It is generated by `Resource` class when constructing. Each `Resource` filter messages according to the `resourceid` property in the message body to ensure that the message is indeed send to this `Resource` and then broadcast them to its `ResourceItem`s.

![img-7](https://raw.githubusercontent.com/privly/privly-organization/master/graphics/diagrams/seamless-posting/img-7.png)

Each Seamless-posting App contains a `appid`, which is used for Privly applications to filter messages sent from content script. It is generated by ResourceItem that creates the iframe (`posting.app.js` or `posting.ttlselect.js`). Again, it is used to ensure that such message is indeed send to this Privly application.

---

The process of showing the Privly button:

#### `posting.service.js`

1. `posting.service.js` adds `click`, `focus`, `blur` event listener to the document of the host page.

2. *(User clicks an editable element)*

3. `posting.service.js` detects whether it is an editable element and calculates the correct target element

4. If there are no `Resource` containing the target element, create one (`createResource@posting.service.js`):

  1. Create ControllerResourceItem implemented in `posting.controller.js`

  2. Create TargetResourceItem implemented in `posting.target.js`

  3. Create ButtonResourceItem implemented in `posting.button.js`

  4. Create TooltipResourceItem implemented in `posting.tooltip.js`

  5. Create TTLSelectResourceItem implemented in `posting.ttlselect.js`

  6. Create a `Resource` containing `ResourceItem`s above and add it to the global `Resource` pool.

5. Send internal message to all `ResourceItem` of the `Resource` that the target element is activated (`posting/internal/targetActivated`)

#### When `posting.controller.js` constructs `ControllerResourceItem`

1. `// Nothing other than add message listeners`

#### When `posting.target.js` constructs `TargetResourceItem`

1. Store the target node in the `ResourceItem`

#### When `posting.button.js` constructs `ButtonResourceItem`

1. Create the DOM node for the button

2. Add event listeners

3. Set the button icon to lock (`updateInternalState`)

#### When `posting.tooltip.js` constructs `TooltipResourceItem`

1. Create DOM for the tooltip (see `posting.floating.js` for underlying implementation)

#### When `posting.ttlselect.js` constructs `TTLSelectResourceItem`

1. Create DOM for the TTLSelect (see `posting.floating.js` for underlying implementation)

#### When `posting.target.js` receives `internal/targetActivated`

1. `posting.target.js` starts the resize monitor (`updateResizeMonitor`) to detect whether the position or size of the target element has changed (`detectResize`).

2. If position or size changed, send internal message to all `ResourceItem` of the `Resource`: `posting/internal/targetPositionChanged`

#### When `posting.target.js` receives `internal/targetDeactivated`

1. `posting.target.js` stops the resize monitor (`updateResizeMonitor`) if this `Resource` is not in `OPEN` state.

#### When `posting.button.js` receives `internal/targetActivated`

1. Update the position of the button (`updatePosition`) according to the position of the target

2. Show the button

3. Set a timer to postpone hiding

#### When `posting.button.js` receives `internal/targetDeactivated`

1. Cancel the postpone timer.

2. Immediately hide the button if it should be hidden according to internal state and `INTERNAL_STATE_PROPERTY`.

#### When `posting.button.js` receives `internal/targetPositionChanged`

1. Updates the position of the button

---

The process of starting seamless-posting or stoping seamless-posting:

#### When `posting.button.js` receives onClick event of the Privly Button DOM

1. Stop if the button is not in a clickable state

2. Send internal message to all `ResourceItem` of the `Resource`: `posting/internal/buttonClicked`

#### When `posting.controller.js` receives `internal/buttonClicked`

1. If the `Resource` is in `CLOSE` state (seamless-posting is not enabled and user clicks the Privly button thus the user is going to enable seamless-posting): Create `AppResourceItem` implemented in `posting.app.js`

2. If the `Resource` is in `OPEN` state: Send internal message to all `ResourceItem` of the `Resource`: `posting/internal/closeRequested` (user has requested to close seamless-posting form)

#### When `posting.app.js` constructs `AppResourceItem`

1. Generate a unique app id

2. Creates an iframe, which `src` is like `privly-applications/Message/seamless.html?contextid=xxx&resid=xxx&appid=xxx`.

3. Append to DOM tree

#### When `posting.app.js` receives `internal/closeRequested`

1. Send message to the app: `posting/app/userClose`

---

The process of hovering on the Privly button:

#### When `posting.button.js` receives onMouseEnter event of the Privly Button DOM

1. Send internal message to all `ResourceItem` of the `Resource`: `posting/internal/buttonMouseEntered`

#### When `posting.controller.js` receives `internal/buttonMouseEntered`

1. Call `tooltip.show` if the `Resource` is in `CLOSE` state

2. Call `ttltooltip.show` if the `Resource` is in `OPEN` state

> There are many other event or message handling process, please check out the code to see others. Event or message handlers above are those typical ones, which I think is enough to give you a comprehensive understanding of the content script architecture.
