## Paginated REST APIs Proposal

The problem: There are many title list views that load many more titles than what is typically found in a "catalog" or "order" title list view. For example, the "Reviews Copies" page in Edelweiss currently loads close to 7,500 titles. Our current model of "load all the titles and associated data at once" that we've leaned on in the react title list view won't work well with this many titles. This is true for 2 reasons: fetching all the personal notes, markup data, tags, product data for this many titles at once is bound to be relatively slow. Second, large catalogs of 1,500+ titles are already displaying warning messages in chrome about high memory usage. This hasn't been a huge concern lately because there are very few catalogs that large, but extending the current react list view to views that are guaranteed to always be 7K+ titles large guarantees to always show this warning.

The proposal: we should implement **server-side in-memory pagination and add support for paginated product data REST APIs**. For example, Edelweiss already loads "all the data at once" for the Reviews Copies page into the server cache. The most straightforward way for us to add pagination to our REST APIs is to layer in "pagination" classes that support sending one "page" of product data to the client at once.

## Propsed API: The 'catalog' API example

Currently, the react list view makes the following API call for catalog and product data:

```
GET: https://qa-api.edelweiss.plus/api/v2/catalogs/4913976/products?productProperties=images,categories,videos,dynamicAttributes,links,audienceRanges,catalogAttributes,completionSummary&apiView=standard,advanced
```
