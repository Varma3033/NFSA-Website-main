# Introduction
‘Australian Film & TV in Time’ is a web app allowing people to browse specifically Australian film and tv culture over the decades (1900s – 2010s) in the National Film and Sound Archive; where each decade is created as a channel on the tv e.g. 1900s = channel 1900. Each channel leads to a collection of film or tv items, and each item page includes images, item title, a director, a summary, and three other items in the channel/decade. The project is purposefully designed to only request items with images, to provide a more visual experience for the user; and there is no search bar, so that they can wander through the app, following the theories of generous interfaces and information flaneur (Dork et al., 2011; Whitelaw, 2015).

To display each page, I wrote all my JavaScript in one index.js file, and each HTML page has a body id which allows the corresponding functions to be called at the bottom of the script. Also, each page is only required to load its content once meaning no JS framework was required.


# Chosen Code Sections

## Fetching and grouping a decade (`index.js`, `getItemImgs()`)
This function allows returned NFSA items to be turned into an object that every page then uses - the object being `{film: [.…], tv: [….]}`. Each item includes a title, year and up to five images using `if (content.images.length < 5) content.images.push(image)`. The channel, collection, item and random pages all call the function, so it practically controls all collection content that is shown on the app. The function is `async` because it uses `await`, meaning it’s paused and only runs until the NFSA has replied/returned results.

```js
//caching - look for saved item data for faster loading times
  const loaded = localStorage.getItem("contents" + year);
  if (loaded) return JSON.parse(loaded);
  
  //web address to fetch
  const url = `${apiurl}?year=${year}-${year+9}&subMedium=Documentation&hasMedia=yes&limit=25`;
  const apiRes = await fetch(url + "&page=1");

 //error state - for actual errors
  if (!apiRes.ok) throw new Error(`Oops! Something went wrong. ${apiRes.status}`);
```

The function is structured to check `localStorage` first and return a decade if it has already been saved to the web browser, resulting in faster loading times (MDN Web Docs, 2026). Originally, my plan was to show all the film and tv items in the NFSA collection, however I found that many returned items had no images resulting in blank or text heavy decades. So, I changed my plan to only show items with images, and I found that by requesting the documentation subMedium (showing stills/posters), I could achieve this, so `const url` asks for `subMedium =Documentation& hasMedia=yes`. I also check if `apiRes.ok` because `fetch()` only throws an error if the request can’t reach the server, so if the NFSA replies back with an error code e.g. 404, it would treat the code as film or tv data (MDN Web Docs, 2025). Hence the check is important to stop my code from doing this, showing the error correctly.

 ```js
 //get other pages of results
  const totalPages = Math.ceil(data.meta.count.total / 25);

  //25 at a time to reduce server load
  for (let start=2; start <= totalPages; start += 25) {
    const requests = [];
    for (let page = start; page < start + 25 && page <= totalPages; page++) {
    requests.push(fetch(url + "&page=" + page));
   }
    //wait for the 25
    const pageRes = await Promise.all(requests);
```

Included in page 1 is `meta.count.total`, telling me the total number of NFSA items. I require all of them as the channel page shows how many film and tv items a decade actually has, and page 1 itself would give a wrong number. So, I used the for loop to send 25 requests together and wait using `Promise.all()`, to reduce NFSA server load (Butler, 2026). 

```js
//add/find img titles to group imgs
    const itemName=item.parentVersion.title || "Unknown";
    let content = contents[contentType].find(img => img.title === itemName);
    if (!content) {
      content = {title: itemName, year: item.productionDates?.[0]?.fromYear ?? 0, type:contentType, images:[]};
      contents[contentType].push(content);
    }

    //allow up to 5 images in item card
    if (content.images.length < 5) content.images.push(image);
  };
  ```

One item in the NFSA is returned per documentation, hence a single film or tv title/entry can have many images linked to it through `parentVersion`. To check if an entry has already been returned, I used `find()`; if it is, the image is added to it; if it doesn’t, a new entry is created.

In terms of alternative approaches there weren’t too many I had considered, however originally I had been requesting `subMedium=film` and `subMedium=tv`, and as mentioned prior most of the returned titles had no images, which didn’t match my idea of having a visually explorable app. Another thing I considered was requesting all the pages at once instead of just 25, but I didn’t want to increase server load and I also didn’t want risk hitting the NFSA rate limit-if they have one-meaning no content would be shown for a period of time (MDN Web Docs, 2025).

Initially I found that an error was being thrown on the second visit of a page. So, I used Claude AI to help me understand why and add what was required. In this instance, an error was showing because `localStorage` only saves strings/text, and originally the browser was storing the literal word “object”, rather than the actual object containing all the page content. So it helped add `JSON.stringify`, turning the object into JSON, and `JSON.parse`, turning the JSON back into an object, solving the error.


## Loading item page (`index.js`, `showContent()`)
The function `showContent()` allows individual item pages to be built and is used in itemcard.html and random.html. It loads stored data first including a title, year, and large image. Since the documentation items don’t have further detail, the function asks for a director, NFSA ID and a summary. I found that splitting the page into two steps, means that the images appear instantly, and only these extra details have to load in, meaning the page doesn’t look completely empty initially. 

```js
//search api by item title & get film/tv item with exact title
    const titleRes = await fetch(`${apiurl}?query=${encodeURIComponent(content.title)}&subMedium=${subMedium}&limit=25`);
    if (!titleRes.ok) throw new Error(`Oops! Something went wrong. ${titleRes.status}`);
    
    //get res as text and then turn into obj - prevent crash
    const textRes = await titleRes.text();
    const data = textRes ? JSON.parse(textRes) : {results:[]};
    const record = data.results.find(item => item.title === content.title);

    //empty state
    if (!record) {
      message(details, "No information.");
      return;
    } 

    //get director
    const director = record.credits?.find(credit => credit.role === "Director");
```

To actually search for a specific item, I first used `encodeURIComponent()`, so any title received including an & or space didn’t break the query string. Then I used `find()` to make sure only records with the exact title name are kept when requested. If there’s no title that matches the item, an empty state is displayed where the text says “No information.”

I used `.text()` to read the received reply before turning it into an object, to make sure the empty state was shown, otherwise `.json()` would just throw an error. To get the directors I used `find()` again to search specifically for the word “Director” in the item’s credits, and if there is no match, an empty state “Unknown" shows again.

One alternative approach I considered was fetching these `showContent()` details in `getItemImgs()`, making the item pages load quickly, but it would also mean an extra request being sent for each title in a decade, many of which may not be opened, unnecessarily slowing down the initial page load and also increasing the chance of an error on the channel page, as if even one fetch fails, the code throws. 

In this section Claude AI did help me troubleshoot an error which I kept getting: “Unexpected end of JSON input”. When using `.json()`, text is expected, and when and empty reply from the NFSA was returned, the function would throw an error. So, AI helped me resolve the issue using .text() and JSON.parse which I explained prior.


## Error Handling and Page States (`index.js`, `message()`, `showError()`, `showChannel()`)

```js
//message function to show state based messages inside element
function message(element, text) {
  element.innerHTML = `<p class="message">${text}</p>`;
}

//function showing error/error message and log to console
function showError(element, error) {
  console.error("Error:", error);
  //error text
  message(element, "Oops! Something went wrong.");
}
```

Each page reads its decade, type and title using `URLSearchParams`, and each page has 4 possible states; loading, empty, error and invalid/edge case. I have written the two functions above to keep consistent state messages across my app. The `message()` function can be used alone to display any loading, empty or invalid state, and it is used within `showError()` to display an error message.


```js
//showChannel()

 //edge case - check decade in page address
  if (!decades.includes(decade)) {
    message(count,"Page not found.");
    return;
}
…
message(grid, "Loading....");
    try {
      const contents = await getItemImgs(decade);

      //film and tv with images count
      count.textContent=`The NFSA has images from ${contents.film.length} films and images from ${contents.tv.length} TV shows in the ${decade}s`;
      
      //film and tv cards
      grid.innerHTML=contentCard("Film", "film", contents.film) + contentCard("TV", "tv", contents.tv);
    } catch (error) {
    
    showError(grid, error);
  }
```

To make sure a page doesn’t crash, I have written code to check whether the decade in the address bar is in the decades array; as anyone can change it and rewrite it e.g. `channel.html?decade=1925`. 1925 doesn’t exist in the array, resulting in a broken page without the initial check. This check, using `message()`, also gives a clear “Page not found” edge case response if the address doesn’t exist.

Each page uses a different element e.g. `showChannel()` uses `grid`, hence why I haven’t handled errors all in a function like `getItemImgs()`. Instead I have thrown them so that each page that calls them can show the message where required. An error reply and network failure end up in the same `catch`, which calls `showError()`, and this logs the error into the console for me along with displaying a message for the user. 

Initially I had the decades 1890s and 2020s included in my app, but after checking responses, I found that they had no items which incorporated images and both were showing empty states, so I removed both of them from my app. This sections code was all written by me.


# Weaknesses and technical debt 
-A single failed page can throw away an entire decade: if any page of results fails in a decade, `getItemImgs()` throws, meaning nothing is shown even if other pages did/would load. But I just simplified the process to make sure the item counts remained correct and due to time restrictions.

-Stored data never expires: Using `localStorage` results in this, and new NFSA data never appears, as stored data is always used. It also can mean `localStorage` runs out of space which results in a page error. I chose not to resolve this due to time restriction and as it never happened during my testing, it was more of a rare case.

-`innerHTML` is used for NFSA text: I created the item page and cards with `innerHTML` as it is easier to read and write. But it can be risky to use, when text is coming from outside my app. It also can break a page if the text contains quotation marks, it won’t be read correctly and would break the `alt` attribute (MDN Web Docs, 2025). 

-Item themes/tags: originally, I planned to have themes and tags on each item page, which the user could click on, and then would be able to view other items in the collection with the same themes, however I ran out of time to find a consistent themes field within each item. 


# Conclusion 
If I kept going, I would fix the `localStorage` data by wrapping the code it in its own try and catch, resulting in the decades page still showing but just not being saved, as it isn’t essential. I would also write some code to be able to skip and run another page if one page failed, even if all the data is not shown it would be beneficial. I would then change to using `createElement()` and `textContent` instead of `innerHTML` for dynamic text (Khan, 2023), to make sure the pages don’t break due to text.


# References
>Anthropic. (2026). Claude (Sonnet 5.5) [Large language model]. https://claude.ai
-Helped with troubleshooting little errors/mistakes in js (as described).
-Helped fix tv frame and navigation alignment, especially when scaling screen size down. 
-Helped with creating placeholder image pattern design.

>Butler, B.E. (2026). Module 3 to Module 6. Canvas. 
https://uclearn.canberra.edu.au/courses/20593/modules

>Dörk, M., Carpendale, S., & Williamson, C. (2011). The information flaneur: A fresh look at 
information seeking. Proceedings of the SIGCHI Conference on Human Factors in Computing Systems, 1215–1224.

>Khan, E. (2023). Replacing innerHTML with createElement. 
https://medium.com/@emrankhandev/replacing-innerhtml-with-createelement-9ba66ad3c2de

>MDN Web Docs (2025, June 11). Rate Limit. 
https://developer.mozilla.org/en-US/docs/Glossary/Rate_limit

>MDN Web Docs (2025, August 20). Using the Fetch API. 
https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch

>MDN Web Docs (2026, October 12). Element: innerHTML property. 
https://developer.mozilla.org/en-US/docs/Web/API/Element/innerHTML

>MDN Web Docs (2026, October 12). Window: localStorage property. 
https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage

>Whitelaw, M. (2015). Generous interfaces for digital cultural collections. Digital Humanities 
Quarterly, 9(1). http://www.digitalhumanities.org/dhq/vol/9/1/000205/000205.html

