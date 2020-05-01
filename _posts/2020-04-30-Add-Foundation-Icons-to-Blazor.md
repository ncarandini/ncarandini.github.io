---
layout: post
title: "Add Foundation Icon Fonts to Blazor"
subtitle: "Because we always need more icons..."
date: 2020-04-30 15:00:00 +0200
background: '/img/posts/P2/background.jpg'
---

Blazor template comes with Bootstrap 4 and OpenIcon icon font, but the number of available icons is less than minimal, so today I've searched for other icon fonts. I won't list them here, as you can easily do it by yourself.

But I wan't to illustrate the steps I've done to use Foundation Icon fonts, a free icon font than I decided to add to my Blazor app.

## Steps

1. Create a folder named "foundation-icons" inside the css folder of your blazor project:

   ![folder creation](/img/posts/P2/001.png)

2. Download the font from the [Foundation Icon Fonts 3](https://zurb.com/playground/foundation-icon-fonts-3) page, clicking on the "Download the Font" button.

3. Unzip the file and copy these files to the "foudation-icons" folder:

   ![folder creation](/img/posts/P2/002.png)

4. Add this style to the foundation-icons.css:

    ```css
    .fi {
        position: relative;
        top: 1px;
        display: inline-block;
        speak: none;
        font-family: Icons;
        font-style: normal;
        font-weight: 500;
        line-height: 1;
        -webkit-font-smoothing: antialiased;
        -moz-osx-font-smoothing: grayscale;
    }
    ```

5. Add this to the top of site.css:

    ```css
    @import url('foundation-icons/foundation-icons.css');
    ```

   And this inside the `.sidebar` class style definition:

   ```css
    .sidebar .fi {
	    width: 2rem;
	    font-size: 1.1rem;
	    vertical-align: text-top;
	    top: -2px;
	}
    ```

6. Now, whenever you like, you can use "foundation icons" and "open icons" in a similar manner:

   ![folder creation](/img/posts/P2/003.png)

## Result

![folder creation](/img/posts/P2/004.png)

That's all folks!