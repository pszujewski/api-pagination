## Paginated REST APIs Proposal

The problem: There are many title list views that load many more titles than what is typically found in a "catalog" or "order" title list view. For example, the "Reviews Copies" page in Edelweiss currently loads close to 7,500 titles. Our current model of "load all the titles and associated data at once" that we've leaned on in the react title list view won't work well with this many titles. This is true for 2 reasons: fetching all the personal notes, markup data, tags, product data for this many titles at once is bound to be relatively slow. Second, large catalogs of 1,500+ titles are already displaying warning messages in chrome about high memory usage. This hasn't been a huge concern as of yet because there are very few catalogs that large, but extending the current react list view to views that are always going to be 7K+ titles large guarantees we will always trigger this warning.

The proposal: implement **server-side in-memory pagination and add support for paginated product data REST APIs**. For example, Edelweiss already loads "all the data at once" for the Reviews Copies page into the server cache. The most straightforward way for us to add pagination to our REST APIs is to layer in "pagination" classes that support sending one "page" of product data to the client at once, where each "page" is 50 titles long.

"Pagination" as a feature requires three states: "filter", "sort" and "page number". The product list must be filtered and sorted before it can be paginated. Currently all this logic is managed in the client. Additionally, "filter" and "sort" require "all the data" in order to work. For example, in order to display all the possible "tag" filters, all the "tag" data must be loaded into the client. **Moving the "filter", "sort" and "page" logic to the server will no longer require the client to load "all the data at once"**.

## Proposed API: The 'catalog' API example

Currently, the react list view makes the following API call for catalog and product data:

```
GET: api/v2/catalogs/4913976/products?productProperties=images,categories,videos,dynamicAttributes,links,audienceRanges,catalogAttributes,completionSummary&apiView=standard,advanced
```

The proposal is that we replace our use of this with the following:

```
POST: api/v1/list-view/products

Request Body: {
    listType: ListType,
    listId?: int,
    productProperties?: List<ProductDTOProperties>,
    apiView?: List<ApiViewOption>,
    page?: int,
    sort?: ProductSortDTO,
    filters?: List<ProductFilterDTO>,
    markupId?: int,
    orderId?: int,
}

// Where 'ListType' uses 'resultTypes', which is defined in TreelineCoreLibrary. For example 'resultTypes.catalog'
// Note: 'listId' is '?' optional because certain "lists" don't require an 'id', such as 'Reviews Copies' page and 'Buzz' page

interface IProductList {
    products: List<ProductDTO>,
    totalAvailableProductsCount: int,
}

Returns: IProductList;

```

Using the same catalog example from above, this would look roughly like if many "filters" were selected:

```
POST: api/v1/list-view/products

Request Body: {
    listId: 4913976,
    listType: ListType.Catalog,
    productProperties: ['images', 'categories', 'videos', 'links', 'dynamicAttributes', 'audienceRanges'],
    apiView: ['standard', 'advanced'],
    markupId: 12345,
    orderId: 678910,
    page: 1,
    sort: {
        sortBy: SortBy.Price,
        sortDirection: SortDirection.Ascending,
    },
    filters: [
        {
            attribute: ProductFilterAttribute.Format,
            value: "Trade Paperback",
        },
        {
            attribute: ProductFilterAttribute.Supplier,
            value: "Flatiron Books",
        },
        {
            attribute: ProductFilterAttribute.Supplier,
            value: "Transit Books",
        },
        {
            attribute: ProductFilterAttribute.Ordered,
            value: false,
        },
        {
            attribute: ProductFilterAttribute.RetailPrice,
            value: "20.00_29.99",
        },
        {
            attribute: ProductFilterAttribute.MarkupTag,
            value: "Frontlist opportunity",
        },
        {
            attribute: ProductFilterAttribute.Category,
            value: "Fiction / Thrillers / Suspense",
        },
    ]
}

Returns: Catalog, which adheres to IProductList;
```

Other "filter" examples:

```
{
    attribute: ProductFilterAttribute.ReviewCopyFormat,
    value: "digital",
},
{
    attribute: ProductFilterAttribute.Note,
    value: false,
},
```

Please see EdelweissComponents "useProductAttributeFilters.types.ts" for more detail and examples. The objects as shown above is close to how the data and state is modeled for the current client-side filtering implementation. It would be fairly simple for the react code to adapt to these changes.

For convenience, here are all the "filter attributes" set up in the client. This is found in the file mentioned above:

```
{
	Supplier = 'supplier',
	Format = 'format',
	Category = 'category',
	Series = 'series',
	PublishingStatus = 'publishingStatus',
	PubDate = 'pubDate',
	RetailPrice = 'retailPrice',
	DiscountCode = 'discountCode',
	Language = 'language',
	Tag = 'tag',
	Note = 'note',
	OrderSuggestion = 'orderSuggestion',
	MarkupPriority = 'markupPriority',
	MarkupNote = 'markupNote',
	MarkupTag = 'markupTag',
	ReviewCopyFormat = 'reviewCopyFormat',
	Review = 'review',
	Honor = 'honor',
	Ordered = 'ordered',
	Lexile = 'lexile',
	ThemaSubjectCategory = 'themaSubjectCategory',
	ThemaSubjectQualifier = 'themaSubjectQualifier',
}
```

## Proposed API: Getting the filter options for a product list

Treeline API would also need to provide with the available filter options for a product list since this data would no longer be available to the client. This is an example of what that would look like:

```
POST: api/v1/list-view/products/filter-options

Request Body: {
    listId: 4913976,
    listType: ListType.Catalog,
    markupId: 12345,
    orderId: 678910,
    attributes: List<ProductFilterAttribute>
}

interface IFilterOption {
    total: int,
    filter: ProductFilterDTO,
}

interface IFilterOptionsByAttribute {
    [attribute: ProductFilterAttribute]: List<IFilterOption>;
}

Returns: IFilterOptionsByAttribute;
```

And here's an example of the shape of the response for clarity using just 2 "filter attributes":

```
{
    [ProductFilterAttribute.Format]: [
        {
            total: 18,
            filter: {
                attribute: ProductFilterAttribute.Format,
                value: "Trade Paperback",
            },
        },
        {
            total: 3,
            filter: {
                attribute: ProductFilterAttribute.Format,
                value: "Hardcover",
            }
        },
        {
            total: 7,
            filter: {
                attribute: ProductFilterAttribute.Format,
                value: "Hardcover with jacket",
            },
        },
    ],
    [ProductFilterAttribute.ReviewCopyFormat]: [
        {
            total: 22,
            filter: {
                attribute: ProductFilterAttribute.ReviewCopyFormat,
                value: "digital",
            }
        },
        {
            total: 13,
            filter: {
                attribute: ProductFilterAttribute.ReviewCopyFormat,
                value: "web reader",
            },
        },
        {
            total: 1,
            filter: {
                attribute: ProductFilterAttribute.ReviewCopyFormat,
                value: "print",
            }
        },
    ]
}
```

## Advantages

Other than the data loading benefits, this propsed update aims to genericize how the react ProductListView loads data for a product list. Rather than needing to extend the "productProperties" etc support to different API Controllers, we can get all the data from a single endpoint. This would allow us to genericize how each ProductListView instance on the client is set up in code as well. This is actually similar to how newLook loads data for a list view with the "GetItemsList" function.

Although the API as outlined above makes no reference to the current "refinements" implementation, the backend code behind this API could certainly leverage "refinements", but that is an implementation detail.
