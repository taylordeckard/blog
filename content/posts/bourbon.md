---
author: Taylor Deckard
title: "Bourbon Search pt. 1: Scraper"
date: 2023-03-11
description: >- 
    A series that discusses building a Whiskey Shop Search with Typescript, MongoDB, GraphQL and Next.js
tags:
    - programming
    - node.js
    - typescript
    - graphql
    - mongodb
    - next.js
    - react
---

Today I'm admitting a guilty pleasure: Bourbon. I read reviews online, hunt for rare bottles, and shell out (maybe a little too much) cash sometimes on interesting things I find. Bourbon has definitely experienced a surge in popularity over the past few years, particularly in the United States. In fact, bourbon has become so popular that the industry is struggling to keep up with demand. The reasons for bourbon's rise in popularity are varied, but some contributing factors include the rise of the craft cocktail movement, a growing interest in American-made products, and an increased focus on artisanal and small-batch products.

One of the consequences of bourbon's popularity is that certain bottles have become highly sought after, and they can command high prices on the secondary market. This is particularly true for limited edition or rare bottlings, as well as bottles that have garnered high ratings from influential critics.

The high demand for these bottles has created a secondary market where collectors and enthusiasts buy and sell bottles at prices far above their retail value. In some cases, these bottles can fetch thousands of dollars, and the prices can be driven up even further by auction houses or online bidding platforms.

Some of the most sought-after bourbons include the Pappy Van Winkle line, which is produced in limited quantities and has a cult following among bourbon aficionados. Other popular bourbons include releases from Buffalo Trace's Antique Collection, such as the George T. Stagg and William Larue Weller bottlings. These bottles can be difficult to find at retail, and some enthusiasts are willing to pay a premium to get their hands on them.

Besides being a fan of whiskey, I'm fascinated by the sudden and explosive growth of markets for popular items. It's intriguing to see how a previously niche product can become a mainstream sensation or a little-known brand can suddenly dominate the market. The study of these sudden market shifts is endlessly captivating, offering insights into the dynamism and unpredictability of consumer behavior and the economy.

---

Thanks to [ChatGPT](https://chat.openai.com/chat) for helping me out with the filler text above. Now let's talk about programming...

# Web Scraper

I wanted an easier way to search for bourbon online. I have a few stores I like to browse and there are so many products that I just don't care about. So I made a web scraper with typescript...

I wanted to design something extensible, since I knew there were several websites I wanted to scrape. For that reason, I started with a `Scraper` base class. Generally the scraping process requires three basic steps:
1. Scrape the total number of products you want to pull from the site.
2. Paginate through the entire list of products and store the product information in memory.
3. Save the data in a database.

Incorporating this idea into the base Scraper class, I started with a simple `scrape` method:
```typescript
export class Scraper {
  public async scrape(): Promise<void> {
    await this._db.connect();
    await this._scrapeAllPages();
    await this._storeData();
    this._db.client.close();
  }
}
```

I created a separate class to manage the DB connection:
```typescript
import { MongoClient } from "mongodb";

export class DB {
  public client: MongoClient;

  constructor() {
    // All credentials are set via environment variables
    const user = process.env.DB_USER;
    const password = process.env.DB_PASSWORD;
    const host = process.env.DB_HOST;
    const port = process.env.DB_PORT;
    const url = `mongodb://${user}:${password}@${host}:${port}?retryWrites=true&w=majority`;
    this.client = new MongoClient(url);
  }

  public connect() {
    /**
     * This will return a promise that resolves after the connection
     * is made. If the client is already connected, the promise
     * immediately resolves.
     */
    return this.client.connect();
  }
}
```
Then imported it into the Scraper.
```typescript
import { DB  } from "./db";

export class Scraper {
  _db: DB;

  constructor() {
    /**
     * The scrapers will be executed in parallel, each within a worker.
     * For this reason each scraper will use a new DB class with separate   
     * connections.
     */
    this._db = new DB();
  }

  public async scrape(): Promise<void> {
    await this._db.connect();
    await this._scrapeAllPages();
    await this._storeData();
    this._db.client.close();
  }
}
```

Ok, good. The database connection is in place. Now to implement step #1.

```typescript
export class Scraper {
  ...
  async _getTotal(): Promise<number> {
    /**
     * This should return the total number products to scrape
     * from the website. This implementation will likely vary for
     * each website, so it just resolves a promise in the base class
     * and the child classes will override.
     */
    return Promise.resolve(0);
  }

  async _scrapeAllPages() {
    /**
     * The first step is to get the number of products to scrape
     */
    const totalProducts = await this._getTotal();
  }
  ...
}
```

Step 2 contains the bulk of the work.

```typescript
export class Scraper {
  _totalPages = 0;
  _productsPerPage = 20;
  _website = "";
  ...
  async _scrapeAllPages() {
    const totalProducts = await this._getTotal();
    /**
     * Add _totalPages and _productsPerPage as protected class variables
     * to be set by the child classes.
     */
    this._totalPages = Math.ceil(totalProducts / this._productsPerPage);
    let currentPage = 1;
    while (currentPage <= this._totalPages) {
      /**
       * Since this is running in a worker, the next line reports progress
       * to the parent process.
       */
      parentPort?.postMessage({
        current: currentPage,
        scraper: this._website,
        total: this._totalPages,
      });
      /**
       * Fetch the webpage, convert text to json, and parse the response.
       * These are methods that can be overridden by the child classes.
       */
      this._parseResponse(
        this._convertTextToJson(await this._fetchUrl(currentPage))
      );
      currentPage += 1;
    }
  }
  ...
}
```
Using the total number of products along with the number of products per page, the total pages are calculated. Then, each page is queried with the `_fetchUrl` method, which looks like this:
```typescript
export class Scraper {
  ...
  _pageKey = "page";
  _queryParams: URLSearchParams = new URLSearchParams();
  _url = "";
  ...
  _buildUrl(page: number): string {
    /**
     * Sets the "page" query param and adds the query params to the
     * website URL.
     */
    const url = new URL(this._url);
    const queryParams = new URLSearchParams(this._queryParams);
    /**
     * Note: The _pageKey is used in case a site uses a different
     * name for the "page" param, like "p" for instance.
     */
    queryParams.set(this._pageKey, String(page));
    url.search = queryParams.toString();
    return url.toString();
  }

  async _fetchUrl(page: number): Promise<string> {
    /**
     * Call the native fetch method to get a page of data. Returns
     * text output (html) to be parsed. Child classes that query
     * an API instead of a webpage can override this method to return
     * another format such as json.
     */
    const response = await fetch(this._buildUrl(page));
    const text = await response.text();
    return text;
  }
  ...
}
```
The fetched page needs to be converted to json (in some cases) and parsed. The base class defines these methods to handle that:
```typescript
export class Scraper {
  ...
  _convertTextToJson(response: string): any {
    try {
      return JSON.parse(response);
    } catch (e) {
      console.error(`Failed to parse json for ${this._website}`);
      return {};
    }
  }

  _parseResponse(response: any): void {
    // Store products in this._data here (implemented in child class)
  }
  ...
}
```
The above `_convertTextToJson` method will only work if the response content is in json form. One of the child classes overrides that method like this, for instance:
```typescript
export class Scraper {
  ...
  _convertTextToJson(response: string): any {
    const $ = cheerio.load(response);
    return $(".main_box")
      .filter((idx, elem) => !$(elem).html()?.includes("sold-out"))
      .map((idx, elem) => ({
        handle: $(elem).find("h5 a").attr("href"),
        price: this._formatPrice(
          $(elem).find(".price .money:first-child").text()
        ),
        title: $(elem).find("h5 a").text(),
      }))
      .toArray();
  }
  ...
}
```
Here, [cheerio](https://github.com/cheeriojs/cheerio) is used to parse html text into an javascript object so that values can be easily extracted by the `_parseResponse` method called next in the flow. Here's an example of that:

```typescript
export class Scraper {
  ...
  _data: Bottle[] = [];
  _productLinkBase = "";
  _scrapeId = 0;
  ...
  _buildLink(handle: string): string {
    return `${this._productLinkBase}${handle}`;
  }

  _parseResponse(response: any): void {
    if (response) {
      response.forEach((product: any) => {
        this._data.push({
          link: this._buildLink(product.handle),
          price: product.price,
          scrapeId: this._scrapeId,
          title: product.title,
          website: this._website,
        });
      });
    }
  }
  ...
}
```
The scraped data is stored in memory to the `_data` class variable. Other class variables used here are
- `_productLinkBase`: Used to build the link to an individual product's page
- `_scrapeId`: See below.

The interface for the stored product data looks like this:
```typescript
export interface Bottle {
  /* Link to the individual product detail page */
  link: string;
  /* Price of the product */
  price: number;
  /**
   * Unique identifier of the current scrape execution. This is used to delete
   * any products from the previous pull that are not present in the current
   * pull.
   */
  scrapeId: number;
  /* Product Title */
  title: string;
  /* Website identifier (used for filtering) */
  website: string;
  /**
   * Flag to denote a product that is present in the latest scrape but 
   * was not present in the previous scrape. 
   */
  fresh?: boolean;
}
```
Now the data is stored in memory and it just needs to be written out to the database:

```typescript
export class Scraper {
  ...
  _storeData() {
    if (!this._data.length) {
	  // Do nothing if no data could be scraped
      return;
    }
    /**
     * Execute a bulk insert operation. If a specific title/website 
     * already exists in the database, the fields will be updated for that
     * document, i.e. "upserted".
     */
    const bulkOp = this._db.client
      .db("bottles")
      .collection("bottles")
      .initializeUnorderedBulkOp();
    this._data.forEach((d) => {
      bulkOp
        .find({
          title: d.title,
          website: d.website,
        })
        .upsert()
        .updateOne({
          $set: d,
          /**
           * On insert, the `fresh` field is set to true. In order for this
           * to work, another step must occur prior to the scrape. I'll explain
           * further down the page.
           */ 
          $setOnInsert: {
            fresh: true,
          },
        });
    });
    return bulkOp.execute();
  }
  ...
}
```
That's enough to build a Scraper. View an example child Scraper class [here](https://github.com/taylordeckard/bottle_search/blob/main/scraper/src/scrapers/cleverwine.ts).

Now to run them in parallel using [workers threads...](https://nodejs.org/api/worker_threads.html) The main `index.ts` file looks like this:

```typescript
import { scrapers } from "./scrapers";
import { DB } from "./db";
import { Worker, isMainThread, workerData } from "node:worker_threads";
import * as cliProgress from "cli-progress";

async function main() {
  if (isMainThread) {
    // On the main thread, all of the child worker threads are spawned.
    const workers = scrapers.map((_, scraperIdx) => {
      return new Worker(__filename, {
        /**
         * The index is passed to the worker so that it knows
         * the Scraper to run
         */
        workerData: { scraperIdx },
      });
    });
    // A set of progress bars is displayed when running via CLI
    const multibar = new cliProgress.MultiBar(
      {
        clearOnComplete: false,
        hideCursor: true,
        format: " {bar} | {filename} | {value}/{total}",
      },
      cliProgress.Presets.shades_grey
    );
    // Bar progress is tracked in a key-value store
    const bars: { [key: string]: cliProgress.Bar } = {};
    // Inialize an array to track when each worker completes its job
    const done = workers.map(() => false);
    workers.forEach((worker, idx) => {
      /**
       * Create a message listener for each worker. Workers report progress
       * after each page is scraped and the bars are updated here accordingly.
       */
      worker.on(
        "message",
        (progress: { scraper: string; current: number; total: number }) => {
          if (!Object.keys(bars).includes(progress.scraper)) {
            bars[progress.scraper] = multibar.create(progress.total, 0, {
              filename: progress.scraper,
            });
          }
          bars[progress.scraper].update(progress.current);
          if (progress.total === progress.current) {
            bars[progress.scraper].stop();
            done[idx] = true;
            // When all workers are done, stop the multibar
            if (done.every((d) => d)) {
              multibar.stop();
            }
          }
        }
      );
    });
  } else {
    // Worker threads hit this branch. 
    new scrapers[workerData.scraperIdx]().scrape();
  }
}

main();
```

The scrapers are exported in the `scrapers/index.ts` file.

```typescript
import { Seelbachs } from "./seelbachs";
import { Sharedpour } from "./sharedpour";
import { Sipwhiskey } from "./sipwhiskey";

export const scrapers = [
  Seelbachs,
  Sharedpour,
  Sipwhiskey,
];
```

## A Few More Things
The finalized `scrape` method looks like this:
```typescript
export class Scraper {
  ...
  public async scrape(): Promise<void> {
    await this._db.connect();
    await this._setScrapeId();
    await this._setFreshness();
    await this._scrapeAllPages();
    await this._storeData();
    await this._deletePreviousScrape();
    this._db.client.close();
  }
  ...
}
```
Notice there are a methods I didn't mention before.

### The `scrapeId`
Every time a scraper runs, it will query the database to select a `scrapeId` from one of the documents. If any documents for the Scraper's `_website` exist, the `_scrapeId` for the current scrape will 1 greater than the `scrapeId` of the document. Otherwise, if there are no documents, the `_scrapeId` for the current scrape is set to 0.
```typescript
export class Scraper {
  ...
  async _setScrapeId() {
    this._scrapeId =
      ((
        await this._db.client
          .db("bottles")
          .collection("bottles")
          .findOne({ website: this._website }, { projection: { scrapeId: 1 } })
      )?.scrapeId ?? -1) + 1;
  }
  ...
}
```
The upsert from the current scrape should update the `scrapeId` for all of the documents. After the newly scraped data is stored, any documents with the old `scrapeId` are deleted. This ensures that if a product was removed from the website, it is also removed from the database.
```typescript
export class Scraper {
  ...
  _deletePreviousScrape() {
    return this._db.client
      .db("bottles")
      .collection("bottles")
      .deleteMany({
        website: this._website,
        scrapeId: {
          $ne: this._scrapeId,
        },
      });
  }
  ...
}
```

### Updating `fresh`ness
Finally, in order to keep track of new products, a flag is added using the mongo `$setOnInsert` operator. However, for this to work, all of the existing documents for the current scrape need to be set to false prior to insertion.
```typescript
export class Scraper {
  ...
  _setFreshness() {
    return this._db.client
      .db("bottles")
      .collection("bottles")
      .updateMany(
        {
          website: this._website,
        },
        {
          $set: {
            fresh: false,
          },
        }
      );
  }
  ...
}
```

## Conclusion
All of the code described here can be found [on github](https://github.com/taylordeckard/bottle_search). In the next post, I'll go over creating a frontend with Next.js and GraphQL.
