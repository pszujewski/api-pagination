## Paginated REST APIs Proposal

The problem: There are many title list views that load many more titles than what is typically found in a "catalog" or "order" title list view. For example, the "Reviews Copies" page in Edelweiss currently loads close to 7,500 titles. Our current model of "load all the titles and associated data at once" that we've leaned on in the react title list view won't work well with this many titles. This is true for 2 reasons: fetching all the personal notes, markup data, tags, product data for this many titles at once is bound to be relatively slow. Second, large catalogs of 1,500+ titles are already displaying warning messages in chrome about high memory usage. This hasn't been a huge concern lately because there are very few catalogs that large, but extending the current react list view to views that are guaranteed to always be 7K+ titles large guarantees to always show this warning.

The proposal: we should implement **server-side in-memory pagination and add support for paginated product data REST APIs**. For example, Edelweiss already loads "all the data at once" for the Reviews Copies page into the server cache. The most straightforward way for us to add pagination to our REST APIs is to layer in "pagination" classes that support sending one "page" of product data to the client at once, where each "page" is 50 titles long.

"Pagination" as a feature requires three states: "filter", "sort" and "page number". The product list must be filtered and sorted before it can be paginated. Currently all this logic is managed in the client. Additionally, "filter" and "sort" require "all the data" in order to work. For example, in order to display all the possible "tag" filters, all the "tag" data must be loaded into the client. **Moving the "filter", "sort" and "page" logic to the server will no longer require the client to load "all the data at once"**.

## Propsed API: The 'catalog' API example

Currently, the react list view makes the following API call for catalog and product data:

```
GET: api/v2/catalogs/4913976/products?productProperties=images,categories,videos,dynamicAttributes,links,audienceRanges,catalogAttributes,completionSummary&apiView=standard,advanced
```

I propose we replace our use of this with the following:

```
POST: api/v1/list-view/products

Request Body: {
    listType: ListType,
    listId?: int,
    productProperties?: List<ProductDTOProperties>,
    apiView?: List<ApiViewOption>,
    page?: int,
    sort?: ProductSortDTO,
    filter?: ProductFilterDTO,
}

// Where 'ListType' uses 'resultTypes', which is defined in TreelineCoreLibrary. For example 'resultTypes.catalog'
// Note: 'listId' is '?' optional because certain "lists" don't require an 'id', such as 'Reviews Copies' page and 'Buzz' page
```

Using the same catalog example from above, this would look roughly like:

```
POST: api/v1/list-view/products

Request Body: {
    listId: 4913976,
    listType: ListType.Catalog,
    productProperties: ['images', 'categories', 'videos', 'links'],
    apiView: ['standard', 'advanced'],
    page: 2,
    sort: {
        sortBy: SortBy.Price,
        sortDirection: SortDirection.Ascending,
    },
    filter: {

    }
}
```
