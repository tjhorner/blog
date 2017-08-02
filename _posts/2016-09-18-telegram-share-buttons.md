---
title: "Telegram Share Buttons!"
layout: post
author: TJ Horner
summary: Telegram has a semi-secret sharing protocol, here's how to implement it as a share button.
permalink: /post/telegram-share-buttons
---

In the [Telegram Blog](https://telegram.org/blog/drafts), there is a "Forward" button
on each post that brings up whatever client you use and prompts you to select a chat.

If you look at the bottom of this post, I've implemented this on my website for
every post. It's really neat and works well as a "share" button!

![](/assets/telegram-share/example.gif)

Here's the code, feel free to steal it:

**HTML:**

```html
<a class="telegram-share" href="javascript:window.open('https://telegram.me/share/url?url='+encodeURIComponent(window.location.href), '_blank')">
  <i></i>
  <span>Telegram</span>
</a>
```

**SCSS:**

```css
.telegram-share{
  position: relative;
  display: inline-block;
  height: 20px;
  padding: 1px 8px 1px 6px;
  font-weight: 500;
  color: #fff;
  cursor: pointer;
  background-color: #0088cc;
  border-radius: 3px;
  box-sizing: border-box;
  text-decoration: none;
  font-size: 11px;
}

.telegram-share span{
  display: inline-block;
  margin-top: 3px;
}

.telegram-share i{
  display: inline-block;
  height: 20px;
  vertical-align: top;
  width: 12px;
  background-repeat: no-repeat;
  background-position-y: 4px;
  background-size: 10px;
  /* This image is included below :) */
  background-image: url(/assets/telegram-plane.png);
}

.telegram-share:hover{
  background-color: #007dbb;
}

.telegram-share:active{
  background-color: #026698;
}
```

**telegram-plane.png:**

![](/assets/telegram-plane.png)
