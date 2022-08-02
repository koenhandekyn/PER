# Infinite scroll with Turbo (HOT)

## Generic considerations

### Avoiding extra count query

In some cases, the total count is not relevant for the user.
In that case, we can avoid a count query.

The crux is to query for `PAGE_SIZE+1` items (one item extra).
If the resultset is larger then PAGE_SIZE, it means that their is a next page.

The assumption is that querying for one extra row is much less overhead
then doing a seperate count query.

## Do we need a pagination GEM?

In the case of infinite scrolling, it feels overkill to depend on one of the
popular pagination GEMs.

The following can be considered as a pattern.
Maybe this pattern can be extracted by means of a small utility class.

```
  # returns
  #   articles: array of articles
  #   next_page: page number of the next page if it exists
  def find_articles
    articles = Article
      .articlenumber(params[:articlenumber])
      .description(params[:description])
      .supplier(params[:supplier])
      .barcode(params[:barcode])
      .price(params[:price])
      .ordered(params[:sort], params[:sortOrder])
      .limit(PAGE_SIZE+1)
      .offset(page*PAGE_SIZE)
      .to_a

    next_page = page+1 if articles.length > PAGE_SIZE
    articles = articles[0...-1]

    return [articles, next_page]
  end
```

## Turbo and ViewComponents

To be able to use the turbo helpers in view components one needs to include
the relevant helper modules into the ViewComponent class.

```
class CustomersNavigationComponent < ViewComponent::Base
  include Turbo::FramesHelper
  include Turbo::Streams::ActionHelper
  ...
```

### ViewComponents CSS handling for GRID based table layouts

The below example shows how one can do a table layout without using HTML table tags.
Also a pattern is presented for doing striped rows.

```
.articles-table {
  display: grid;
  grid-template-columns: minmax(0, 2fr) minmax(0, 2fr) minmax(0, 8fr) minmax(0, 2fr) minmax(0, 1fr) 3em;
  max-width: 100%;
  width: 100%;
}

.articles-table .col {
  height: 2.75rem;
  display: flex;
  align-items: center;
  width: 100%;
  padding-left: 0.5rem;
  padding-right: 0.5rem;
}

.articles-table .align-right {
  justify-content: right;
}

.articles-table .col-striped {}

.articles-table .col-striped:nth-child(12n+7),
.articles-table .col-striped:nth-child(12n+8),
.articles-table .col-striped:nth-child(12n+9),
.articles-table .col-striped:nth-child(12n+10),
.articles-table .col-striped:nth-child(12n+11),
.articles-table .col-striped:nth-child(12n+12) {
  background-color: #EEEEEE;
}
```

the related HTML would look like this

```
.articles-table
  - articles.each do |article|
    .col.col-striped = article.articlenumber
    .col.col-striped = article.barcode
    .col.col-striped = article.description
    .col.col-striped = article.supplier
    .col.col-striped.align-right = article.price.print
    .col.col-striped.align-right
      a.btn.btn-sm.btn-secondary href=edit_article_path(article)
        i.bi.bi-pencil
```

With viewcomponents the code can be organized like this

- articles_page_component.html.slim
- articles_page_component.rb
- articles_page_component.scss

where the view components scss is imported into the main application scss for
efficient delivery to the client

```
@import 'articles_page_component';
```

## Variant 1

The principle in this variant is that the next page is part of the current page
and lazy loaded when it becomes visible as a turbo frame.

Because HTML tables are very strict on which elements can be contained inside
a table, this solution forces an alternative way for rendering the content as a
table. CSS grid is a valid alternative.

A minor performance improvement was added, i.e. the index page can be queried
in raw format in which case it will not apply the layout and purely provide
the new rows in HTML format.

### PRO

- simple and purely based on Turbo Frame's

### CON

- does not work with HTML tables
- resulting DOM not clean

### Example code

articles_page_component.html.slim

```
= turbo_frame_tag "articles_#{page}" do
  .articles-table

    - articles.each do |article|
      = render(ArticleRowComponent.new(article:))

    // same like (and similar performance, not like with partials)
    // = render(ArticleRowComponent.with_collection(articles))

  - if next_page
    = turbo_frame_tag "articles_#{next_page}", :src => next_page_raw_path, loading: "lazy" do
      .articles-table-loading
        p
          | Click
          a<> href=next_page_path data-turbo-frame="_top" here
          | to go to the next page.
```
articles_page_component.rb

```
class ArticlesPageComponent < ViewComponent::Base
  include Turbo::FramesHelper

  attr_reader :page, :articles, :next_page

  def initialize(articles:, page:, next_page:)
    @articles = articles
    @page = page
    @next_page = next_page
  end

  def query_params
    request.query_parameters.except(:page, :raw)
  end

  def next_page_path
    articles_path(page: next_page, **query_params)
  end

  def next_page_raw_path
    articles_path(raw: true, page: next_page, **query_params)
  end
end

```

articles_controller.rb

```
  ...
  def index
    articles, next_page = find_articles
    respond_to do |format|
      format.html {
        if !params[:raw]
          render locals: { articles: articles, page: page, next_page: next_page }
        else
          render(ArticlesPageComponent.new(articles: articles, page: page, next_page: next_page), layout: nil)
        end
      }
    end
  end
  ...
```

articles/index.html.slim

```
= render(ArticlesPageComponent.new(articles:, page:, next_page:))
```

## Variant 2

The approach in this variant is that we have a section in the code that contains
the table data (HTML table or as CSS grid).

Below it we have a lazy loaded turbo frame that does the following when it becomes
visible by linking it to a turbo stream request (this frame is called the navigation
frame)

- append the next page of data/rows to the table data section
- replace the navigation frame with an updated version that does the same for the next page,
  when a next page is available.

### PRO

- resulting DOM is cleaner
- possibly a bit more generic in it's application

### CON

- a bit more code that uses Turbo Stream capabilties in combination with Turbo Frame

### Example code

customers_nagivation_component.rb

```
# frozen_string_literal: true

class CustomersNavigationComponent < ViewComponent::Base
  include Turbo::FramesHelper

  attr_reader :next_page

  def initialize(next_page:)
    @next_page = next_page
  end

  def query_params
    request.query_parameters.except(:page, :raw, :format)
  end

  def next_page_path
    customers_path(page: next_page, **query_params)
  end

  def next_page_stream_path
    customers_path(format: :turbo_stream, page: next_page, **query_params)
  end
end
```

customers_navigation_component.html.slim

```
- if next_page
  = turbo_frame_tag "customers-table-navigation", :src => next_page_stream_path, loading: "lazy", target: "_top"  do
    .customers-table-loading
      p
        | Click
        a<> href=next_page_path data-turbo-frame="_top" here
        | to go to the next page.

```

customers_page_component.rb

```
class CustomersPageComponent < ViewComponent::Base
  attr_reader :customers, :next_page

  def initialize(customers:, next_page:)
    @customers = customers
    @next_page = next_page
  end
end
```

customers_page_component.html.slim

```
#customers.customers-table
  - customers.each do |customer|
    = render(CustomerRowComponent.new(customer:))

= render(CustomersNavigationComponent.new(next_page:))
```

customers_controller.rb

```
...
  # GET /customers
  def index
    customers, next_page = find_customers
    respond_to do |format|
      format.html { render locals: { customers: customers, page: page, next_page: next_page } }
      format.turbo_stream { render locals: { customers: customers, page: page, next_page: next_page } }
    end
  end
...
```

customers/index.html.slim

```
= render(CustomersPageComponent.new(customers:, next_page:))

```

index.turbo_stream.slim

```
= turbo_stream.append(:customers)
  - customers.each do |customer|
    = render(CustomerRowComponent.new(customer: customer))

= turbo_stream.replace(:"customers-table-navigation")
  = render(CustomersNavigationComponent.new(next_page:))
```
