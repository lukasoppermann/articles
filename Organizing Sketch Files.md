# Organising Sketch Files

As designers we often start a new sketch document without any thought and let it grow however it will. While this is mostly fine for you when you work on this file every day, having a wildly grown file can be problematic for others who might work with your file like developers, other designers or yourself in a month from now.

In this article I explain what I found to be the most efficient way of structuring & organising your files to make working with them easy and enjoyable.

Although I focus on sketch files, most of the ideas are applicable for any type of design file. However some ideas might not work as well if your tool does not support pages.

## Naming
Naming your files well is not hard, if you know what to consider:

1. The file might be shared and stored out of context (e.g. not in the project folder)
2. There might be different design aspects to the project that require individual files

With those points in mind, names like `design.sketch`,  `app.sketch` or `myApp.sketch` are all bad choices.

However by combining the project name and the platform you get a clear and understandable naming conventions. Using this convention with an app called `projectManager` your files would be named `projectManager_ios.sketch`, `projectManager_android.sketch` and `projectManager_web.sketch`. Starting with the project is helpful because it lets you sort by project.

> The important takeaway is to specify the **project** as well as the **platform** in the file name, so that you can differentiate between different platforms and know what content the file has, even if it is attached to an email or somewhere on a usb stick without any context.

## Versioning
With the naming convention figured out, we can move on to one of the most important aspects: **versioning**.

How often did you accidentally delete something you needed later on, had a corrupted file and lost a whole week worth of work or, to prevent the first, cluttered your file with variations and potential features which you wanted to keep, just in case. Versioning can solve all of those problems with ease.

### Version Control System

One fairly new concept for designers is the use of a [version control system (vcs)](https://en.wikipedia.org/wiki/Version_control). While you could use git before (and I confirm that it works quit well), it has been a bit of a hassle. Services like [Abstract](https://goabstract.com) promise to solve this with a visual representation of your sketch file as well as an integration which lets you quickly commit a saved change to the vcs. This means that you can keep the same file name and, in case you need something from before or your file gets corrupted, you should be able to get it back rather easily.   

Additionally a vcs allows you to *branch out*, meaning you create a separate version that lives side-by-side with your original version (`master` branch). In programming this is used to develop a new feature or to test something and if successful, the feature branch can be *merged* into the `master ` branch. I have not really tried this concept with sketch files, I use it all the time for coding, but I am not sure it makes sense for design. Time will tell.

### Date and Name

What I have seen a lot, especially in agencies, is to prefix or suffix files with a todays date, e.g. `YYYYMMDD_project_name` or `project_name_YYYYMMDD`. This has the benefit of being sortable by day and theoretically, if you never forget to copy and rename the file, you cannot lose more than one day of work.

> However, I feel this approach has more problems than benefits and I would strongly recommend against it!

One problem I see with this convention is that losing a whole day of work might be quite a lot and if you create some versions of a screen and want to clean you file but keep those versions save, you are at a loss. So you probably start appending a `2`, `3`, etc., hacking a naming convention is a red flag and always means there is an issue with it.

Another problem arises when you add a tiny changed. You want to save this file, but if you create a new version, people are going to be confused because they cannot see the visual difference in the two versions. However if you save the file with the old name you will have a file that has a date in it, which does not align with the last modified date.

### Date Labeled Pages

A slightly different, and even worse take on using the date, is to label the pages within sketch with todays date. Apart from the aforementioned problems, this approach raises some additionally issues.

Duplicating the page every day means you grow you sketch files size at an unnaturally fast rate, which will lead to slow opening times.

With this approach you also made sure that if your file gets corrupted or deleted, you will lose everything you ever worked on, because you have only a single file.

Further more, using pages as versions means you can not use pages to separate screens or sections without creating a giant mess. You basically make you files structure worse.

> The short summary is: Don't even think about this approach, it is horrible.

### Name and Continuous Number

The by far best naming convention (apart from using a vcs system) is to suffix your file with a continuous number. Personally I like to use a dot-separated number `##.#` so my file name would be `projectManager_mac_01.7.sketch`. However I see no issue with using integers, I would just recommend to use a leading zero, so that the files are sorted properly and `20` follows `19` and not `2`. Should you regularly exceed `99` versions, use two leading zeros.

The benefit of this approach is that you can version as often or as little as you need. Sometimes you might create 8 versions on one day, or work on the same version for an entire week, because not much changes. Also this approach does not mess with your files structure, file size or naming convention. I recommend to copy the current version into a `history` or `versions` folder in your project and rename the new version, so that you always only have the current version in your projects root folder.

### Collaborating on a file
There might be situations where two people need to work on one file. If two people work in one file simultaneously, you need to use a vcs, no question about it. However if you work on a file and hand it off to a colleague and maybe work on it again some time later and hand it off again, I have found it most useful to suffix the version with a short form of the persons name, so that you have no chance of ever overwriting somebody else's file. We used this approach a lot when I worked for an infographics agency and never had any issues. Using this approach, my file would be named `projectManager_mac_01.7_lo.sketch`.

## Organising files

The most important part to make sketch files easy to use for others and yourself is of course the structure within the file itself. Over time I have developed some methods I find particularly helpful. However, depending on the project I sometimes only use some parts, especially on very small projects.

### Styleguide
A style guide is something I try to use in every project. However this style guide is not supposed to be an extensive guide that explains every last detail. Rather it is supposed to define and show the main design decisions that have been made, so that anybody who works with this file can quickly see which rules are defined.

I recommend to label the first page in a sketch file "Styleguide" and create separate artboards to show:

- all colors that may be used
- in case gradients are used regularly, those should be on a separate artboard in the style guide
- all typefaces, specific fonts (e.g. "Gotham HTF Bold") and the most common sizes (Title, H1, H2, H3, paragraph, etc.)
- sizes that should be used: I like to define a modular scale and use only sizes within this scale. The artboard shows all sizes, so it is a quick reference for available sizes.

### Wireframes
When creating an entire application or a complex feature from scratch, it is a good idea to create a wireframe first and discuss its with the team, to avoid wasting effort on designing something that can't be build.

When designing you will refer back to the wireframe frequently to make sure you have not missed anything that is required on the screen you are currently designing, as the wireframe is like a user story for a developer. To make this easy, I find it best to keep the wireframe in your sketch file. I recommend to use the second page of your sketch file for the wireframe.

> A wireframe in your sketch file makes it much easier for a new designer to understand the file and what you are working on.

### Designs
Now we finally get to the meat of your file, the designs.
There are a myriad of ways to structure your pages but I found the following quite pleasing and good to work with.

#### Userflow

I have found it most useful to separate my designs into sections, mostly feature related, and give every feature a unique number. For example, my first page after the `wireframe` page could be `01 Onboarding`.

On this page I have an artboard showing the userflow for the given feature. I use a standardised, simplified wireframe like style to show how the user moves through the screens in this section. I recommend creating a library with elements to quickly create user flows. The benefit of having this in the beginning of every section is that one can quickly grasp what this feature is about and how the screens behave. This is very helpful for explaining something to a stakeholder, but also for a developer working with your files.

Every screen in the user flow gets a unique number. The `login` screen in our example would maybe be `01.2.0` because it is part of `01 Onboarding` and within this it is the second part (part one being the intro slides) and the first screen of this part.
The `wrong user or password error` page could be `01.2.1` and so on. This makes it much easier to reference screens when talking about them via Skype or in tickets.

#### Screen Designs

After the userflow page I create subpages for every feature of the section. While you cannot create real subpages, you can visually do so by prefixing the page name with the `↳` character. This means my pages sidebar looks like this now:

```
◆ Styleguide
◆ Wireframes
01 Onboarding
↳ Intro
↳ Login & Register
```

I find this visually much more organised than showing the actual screen numbers and in some sections, like `Login & Register`, I would have multiple sub-features with different numbers.

### Layers

Going one level deeper, into pages and dartboards, there is another important aspect of organisation: your layers.

#### Clean up Your Layers

One quick and sensible thing is to remove all empty layers as well as unnecessary layers. While it is fine to have multiple images in a group to test out which one works best, once the best one is found all others should be deleted. The same holds true for any other element. You should not have any hidden layers in your file, ever. Hiding is just a tool while working on something, when you close your file all layers should be visible.

#### Naming Layers

Make sure to pick a sensible name for your layers, as automatically created names like `Rectangle 10` or `Oval Copy` are not very helpful. Sometimes you compose a visual element from multiple layers, if you don't have to many layers, it is enough to name the group. This will make sure that future interactions with your files are pleasant and not a horrible waste of time.

Naming layers should be a no-brainer. Use one language (preferably the language everybody in your team speaks) and name the layers for their function. E.g. name a profile picture `profile picture` and not `Mary` (even though your screen shows your persona called Mary).

A nice touch is to add emojicons to your layers, to easily identify important layers, e.g. `🌄 profile picture`. You can easily bring up the emojicon picker by pressing `ctrl` + `cmd` + `space`. Don't overdo it though.

#### Symbols

One extremely effective instrument for organising your files are symbols in sketch.

> Symbols are reusable components that share a common design, but can have some flexible parts.

Whenever you have a reusable object or a complex object it makes sense to create a symbol from it. For example if you create a symbol for a list item, you can later update the text color and it will change for all instances of the symbol.

However, sometimes you might want to have some flexibility to it, for example you might want some list items to have an arrow on the right and others not. To make this possible, you need to nest a **symbol** of the arrow into the symbol of the list item. Now whenever you place the list item symbol in your file you will see a dropdown on the right sidebar in sketch and you can select `none` to hide the arrow.

You can also edit images and text in this sidebar. However, sometimes an image or text should not be changed. To hide any item from the sidebar, simply lock it in the symbol.

One very important part of creating good symbols is naming the layers correctly. There are two reasons for this.

Firstly when you have two different button symbols and they both have a text layer named `label`, you can switch the symbols via the dropdown and the overwrite text will be kept.

Secondly the names of your layers are used for the labels in the sidebar. Especially when nesting symbols more deeply this can make everything much less confusion. I recommend to use a visual indicator to denote symbols as children, just like with subpages. So your list item might have a text layer named `↳ label`. Whenever I put a  symbol with nested symbols into a bigger symbol, for example when nesting the list item into a group if multiple items with the same name, I rename the symbols from something like `list item` to `◆ list item`. This makes it much easer to see where a new items options start, within the sidebar.

### Libraries

Libraries in sketch are a great tool to share symbols across multiple files or projects. However, it does make the whole editing flow more complicated, so I would recommend to use libraries with care.

Some very good scenarios for when to use libraries are:

- Wireframes: A library with common elements, like screens, lists, buttons and arrows. Since wireframes visually can look the same in every project it is easy to share those elements.

- User flows: Similar to wireframes user flows can share a common visual aesthetic across multiple projects and can be kept fairly general.

- Icons: For projects where icons are used but do not contribute much to the distinctive visual style, a shared set of icons from a library can be used.

- Sharing elements across a products: When you work on a product which caters to different plattforms or where different parts are split into different sketch files due to size, extracting shared elements like buttons into a library can be very helpful.
