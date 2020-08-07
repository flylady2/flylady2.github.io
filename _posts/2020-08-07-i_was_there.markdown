---
layout: post
title:      "I Was There"
date:       2020-08-07 21:05:54 +0000
permalink:  i_was_there
---

For my Rails/JS project I made an online diary called “I Was There” (https://github.com/flylady2/i-was-there-frontend).  I started the project during the covid-19 pandemic when, like many non-essential workers, I was spending a lot of time at home and the days were starting to bleed into one another.  I wanted to create something that would force me to be “more present” and to pay attention to what was around me, and that would also encourage me to be positive and productive despite the pandemic crisis.  My diary format is simple.  The unit is a day.  Each day has a name (day of the week) and date attributes, as well as one image with a caption, and 6 entries with content. The entries must be associated with a category that is selected from a drop down menu that includes choices such as What I’m Reading, Lesson Learned, Problem Solved, In the Garden, Something Nice, etc. 

**About the name.** When I was in school (a long time ago but in this galaxy), we were shown historical films from the CBS “You Are There” Series that depicted reenactments of famous historical events.  The film would set the scene and then the narrator would proclaim, “and YOU are there!” and I would get a little shiver of excitement.  Because this app is intended to encourage me to be more present in my life, to “be there”, I decided to name it after those fondly remembered films. 

**About the process.** While working on the project I kept a journal in which I described each day’s coding progress.  Reading over my journal I saw that almost every day I struggled with some kind of challenge, but I also realized that the “figuring it out” part is what I enjoy the most about coding.  The times I was most excited about getting started working were the ones that followed a session in which I had discovered a problem but had not yet solved it.  Sometimes it was hard to stop working before I had figured it out.  But after a certain point I knew I should, because I often found that the solution, or an idea about how to approach the problem, would come to me in between coding sessions, when I was doing something completely unrelated.

**About the code.** Some of the example apps I’d seen before starting my project show all of the unit items in the database upon page load, e.g. a Notes app that shows all of the notes in the database.  Since I ultimately hope to have a database with many days, each of which has an image and six entries, it wasn’t practical to display them all.  Instead, when it is first opened, the app displays only the most recently submitted day in the database, `Day.all.last`.

When a user creates a new day, or searches for a day by date, the data for the new/found day replaces that of the day that was displayed upon page load.  Because the new/found day is replacing the current day, I had to figure out a way to do that without having any of the current day’s data, like its id.  For the day’s attributes of name and date, and for the image and its caption, this was simple - I use `document.getElementById()` to pull up the specific elements containing those items and replace their contents with the new day's and image's attributes.

For the entries, however, there are six of them per day and I needed a way to easily match up a new and old entry such that all the old entries are replaced by new ones.   The entries and their associated data are contained within the `newEntriesData` array that is extracted from the json as `day.included`.   I use a `for loop` to iterate through the `newEntriesData` array, assigning each entry an “i" number based on its position in the array. The “i”  number, along with the entry’s attributes, are passed to the Entry constructor to create a new Entry instance.

```
const newEntriesData = day.included
for (let i = 0; i < newEntriesData.length; i++) {
let newDayEntry = new Entry(newEntriesData[i], `${i}`).renderEntry()
}
```


```
class Entry {
  constructor(entry, i) {
    this.id = entry.id
    this.category_name = entry.attributes.category.name
    this.content = entry.attributes.content
    this.i = i
  }
```
When entries for the day that is displayed upon page load are rendered, each one is contained within a newly created `div` that is assigned the entry's “i” number as its `id`.

```
const divCol = document.createElement('div’)
divCol.setAttribute('id', this.i)
```

When a new or found day is created, the”i” number that is assigned to each entry is used to pull up the `div` with that `id` and the `div` is re-populated with the new entry’s attributes.
```

const divCol = document.getElementById(this.i)
```
One consequence of structuring the code this way is that if I don’t have an image and all six entries, it really messes things up!  To avoid that problem, all of the input fields on the create day form are required and the “Create new day" submit button does not work unless they are all filled out.  Although this might be annoying for another user, this feature was an advantage for me because my intention was to force myself to generate an image and six entries for each day. 
 
Another notable feature is that only entries for newly added days can be edited.  It’s not possible, for example, to pull up a day from two weeks ago and change the entries to something else, something….better.  That would be cheating!  But if the user submits a new day and when it is displayed they realize that they forgot something or included a typo, they can edit an entry’s content and update it.

Since I only want to allow entries for a new day to be editable, edit buttons appear only in the cards containing entries for a newly created day. When an edit button is clicked, the `makeEditable` function is called, which makes the entry’s content editable and generates a submit button. If a change is submitted the process can be repeated, but once the page is refreshed the newly created day is rendered on the basis of being the most recently created day in the database, which cannot be updated.

I originally built everything out without using Object Oriented Javascript and then refactored to create `Day`, `Image` and `Entry` classes.  I moved all of the rendering functions for day, image and entry into their respective files.  But when I moved the `makeEditable` method into `entry.js` I got an error.  I was stumped since the exact same code was working perfectly in my `index.js` file.  None of the debugging helped until I finally had a flash of insight (or memory) that if `makeEditable` is a method in the `Entry` class, then it needs to be called on an instance of the `Entry` class.  That however, presented a problem.  The only information this method receives is the entry’s `id` and` content`.  The method does not collect the name of the category the entry belongs to nor its “i” number, which are required data to create a new `Entry` instance. 

Still, I wanted to remove the two functions associated with editing entries, `makeEditable` and `renderEditedEntry`, out of the `index.js` file and group them together in a separate file.  I initially considered trying to make a module containing these two methods.  Although I learned a fair amount about modules while trying to figure out how to use them, this approach was unsuccessful.  First, it was not practical to have just one module and all the other class files not be modules.  More importantly, I was operating under the mistaken assumption that variables declared in the module would have a global scope.  (I’m sure I read that somewhere, because that was why I thought a module was the way to go.)   However, they do not.  According to [https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules ](http://), “Last but not least, let's make this clear — module features are imported into the scope of a single script — they aren't available in the global scope.”

So I decided to instead create an `EditableEntry` class which only has `id` and `content` attributes. When the edit button is clicked, a new` EditableEntry` instance is created and the `makeEditable` function is called on it.

```
   editBtn.addEventListener("click", event => {
      let newEditableEntry = new EditableEntry(event.target.id, p.innerText).makeEditable()
      })
```

This makes its content editable and creates a submit button.  

 ```
class EditableEntry {
  constructor(id, content) {
    this.id = id
    this.content = content
  }

   makeEditable() {
    let p = document.getElementById(this.id)
    p.contentEditable = true
    let div = p.parentElement
    let submitBtn = document.createElement('button')
    submitBtn.setAttribute('id', `${this.id}`)
    submitBtn.textContent = 'Submit'
    div.append(submitBtn)
    submitBtn.addEventListener("click", event => {
      editEntry(event.target.id, p)
   })
  }
```

Clicking the submit button calls the `editEntry` function in` index.js`, which makes the patch request to the database.  The returned `json` is used to create a new `EditableEntry` instance that `renderEditedEntry` is called on. That method replaces the inner text of the p element containing the entry's content.  This approach allowed me to have all the render methods in the various class files and only the fetch request and form handling methods in `index.js`.

I had a lot of fun building this project despite and *because* of all the challenges.  Many thanks to Flatiron instructor Ayana Cotton for her Rails/JS Project Build study groups.  They were immensely helpful!




