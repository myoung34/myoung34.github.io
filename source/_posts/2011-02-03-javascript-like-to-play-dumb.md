---
title: JavaScript Likes To Play Dumb
author: myoung
excerpt: Coercion = Fun time
layout: post
comments: true
permalink: /post/javascript-like-to-play-dumb/
categories:
  - JavaScript
  - Programming
tags:
  - bug
  - javascript
  - programming
  - rant
---
My first post is going to be some fun coersion issues I have found lately while touching up on my JavaScript

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="javascript" style="font-family:monospace;"><span style="color: #CC0000;"></span> <span style="color: #339933;">==</span> <span style="color: #3366CC;">''</span>   <span style="color: #006600; font-style: italic;">// returns true</span>
<span style="color: #CC0000;"></span> <span style="color: #339933;">==</span> <span style="color: #3366CC;">'0'</span> <span style="color: #006600; font-style: italic;">//returns true (odd since false == 'false' returns false)</span></pre>
      </td>
    </tr>
  </table>
</div>

&#8230;And&#8230;

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="javascript" style="font-family:monospace;"><span style="color: #3366CC;">' '</span> <span style="color: #339933;">==</span> <span style="color: #CC0000;"></span> <span style="color: #006600; font-style: italic;">// returns true</span></pre>
      </td>
    </tr>
  </table>
</div>

Also my favorite (thanks to destroyallsoftware)

<div class="wp_syntax">
  <table>
    <tr>
      <td class="code">
        <pre class="javascript" style="font-family:monospace;"><span style="">Array</span><span style="color: #009900;">&#40;</span><span style="color: #CC0000;">5</span><span style="color: #009900;">&#41;</span>.<span style="color: #660066;">join</span><span style="color: #009900;">&#40;</span><span style="color: #3366CC;">'wat'</span><span style="color: #339933;">+</span><span style="color: #CC0000;">1</span><span style="color: #009900;">&#41;</span><span style="color: #006600; font-style: italic;">//returns wat1wat1wat1wat1wat1</span>
<span style="">Array</span><span style="color: #009900;">&#40;</span><span style="color: #CC0000;">5</span><span style="color: #009900;">&#41;</span>.<span style="color: #660066;">join</span><span style="color: #009900;">&#40;</span><span style="color: #3366CC;">'wat'</span><span style="color: #339933;">-</span><span style="color: #CC0000;">1</span><span style="color: #009900;">&#41;</span><span style="color: #006600; font-style: italic;">//returns NaNNaNNaNNaNNaN</span></pre>
      </td>
    </tr>
  </table>
</div>
