# Cross Browser Emoji in ContentEditable

Emoji support cross browser is [not good](http://caniemoji.com).  Chrome doesn't support them on any platforms, and
most versions of Windows and Linux don't have a color emoji font to begin with, so if we want to support emoji on the
web (a feature that users increasingly demand) we need a workaround.  There are a number of JavaScript libraries out
there that support cross browser emoji by replacing emoji characters with images.  However, I hadn't seen any that
tried to make them work inside contenteditable.

ContentEditable is used by many sites around the web to implement rich text editors.  Everyone
who has used it knows it's [terrible](https://medium.com/medium-eng/why-contenteditable-is-terrible-122d8a40e480), but
we use it anyway since there are no alternatives.  To find out just how terrible it is, let's try to get emoji working!

## Try #1: Image tags

The first thing I tried, was to just use one of the many JS libraries out there to convert emoji to image tags inside
contenteditable and be done with it.  This has some major problems:

1. Image elements get these ugly resize handles in IE and Firefox when you click on them, and there's no way to remove them.
2. Image elements don't work like inline elements with text selections, cursor movement, etc.
3. You can drag and drop images around, unlike text.
4. Copy and paste doesn't work right.
5. etc.

Here's an image of #1:

![](https://i.cloudup.com/bxzAnpXX_T.png)

## Try #2: Inline spans

The next approach to try was to use `<span>` elements with background images.  This works better since the elements are inline,
but there are problems with this approach too (which we'll fix).  We have to put the actual emoji character inside the span element
so that copy and paste works right, and we have some width for the background image.  This causes the actual character (or an ugly
black missing glyph) to appear above the background image.  This seems like it would be simple to solve by setting `color: transparent`
in CSS, but unfortunately the color property also controls the color of the cursor in contenteditable, and an invisible cursor is not
very desirable.  In WebKit-based browsers, there is a `-webkit-text-fill-color` property that works when set to `transparent` and does
not effect the cursor, but other browsers don't have such a property.  So, we're back to square one.

Another problem with this approach is that the widths can be wrong for some characters.  Some emoji are actually two or more characters
that are combined by the font engine into a ligature.  Examples include the flag glyphs (e.g. :us:), and the keycap glyphs (e.g. :one:).
Since the span elements are inline, we can't override their intrinsic widths.  We could try to set the span element to `display: inline-block`,
but this causes the same problems as with images.  In IE at least, the resize handles show up and text selections don't work right
for all block elements inside contenteditable, not just images.

Here's an image showing both problems:

![](https://i.cloudup.com/OuSMATLvVH.png)

## Solution: Inline spans with custom font

The solution that I came to that solves both of these problems was to create a custom web font that contains empty (invisible) glyphs
of the correct widths.  This way, the spans can be inline elements and we don't have the problem of the glyphs being drawn above the
background images since the glyphs being used in this font are empty, and we don't have the width problem with multiple characters
since the glyphs in the font can be designed to the correct widths.

This font requires just three glyphs, and a mapping of all of the supported emoji characters to one of these glyphs.  The first glyph
is just a normal width glyph used for most single characters.  The second is a half width glyph used for the flag glyphs, which contain
two characters each.  The third is a zero width glyph used for U+20E3 COMBINING ENCLOSING KEYCAP and U+FE0F VARIATION SELECTOR-16.  We
want those to be zero width so that the prior glyph that it modifies gets all the space.

In order to actually make this font, I used a tool called [fonttools/ttx](https://github.com/behdad/fonttools/).  This allows generating
a font file from an XML file.  I've included the XML source code for the font in this repo, and you can check it out for yourself if you're
interested.  You can generate a font file using the following command:

    ttx emoji.ttx

Once we have this font, things are pretty simple to set up in CSS.

```css
@font-face {
  font-family: Emoji;
  src: url('emoji.woff2') format('woff2'),
     url('emoji.woff') format('woff'),
     url('emoji.ttf') format('truetype');
}

.emoji {
  font-family: Emoji;
  background-size: auto 1em;
  background-repeat: no-repeat;
  background-position: top center;
  vertical-align: top;
}

```

The JS is a little more finicky just because of how terrible contenteditable is, and it really depends on how the rest of your editor is
set up as to how you do it.  You can see a terrible hacky way to do it (replace the HTML after every keystroke) in the
[demo](https://devongovett.github.io/contenteditable-emoji/).  Each span should look like this after the replacement:

```html
<span class="emoji" style="background-image: url(d83d-dc4d.png)">👍</span>
```

This technique seems to work pretty well cross browser (tested in IE10, Firefox, Safari, and Chrome).  Definitely check out the
[demo](https://devongovett.github.io/contenteditable-emoji/) for yourself and let me know what you think.
