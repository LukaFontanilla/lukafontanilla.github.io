# eCommerce Walkthrough: An Intro to Malloy Syntax & the VSCode Plugin
_The following walk-through covers similar concepts to the [Quick Start](https://github.com/looker-open-source/malloy/#quick-start-videos) videos in the README. You can find the complete source code for this model [here](https://github.com/looker-open-source/malloy/blob/docs-release/samples/ecommerce/ecommerce.malloy)._

Malloy queries compile to SQL. As Malloy queries become more complex, the SQL complexity expands dramatically, while the Malloy query remains concise and easier to read.

Let’s illustrate this by asking a straightforward question of a simple ecommerce dataset--how many order items have we sold, broken down by their current status?

```malloy
--! {"isRunnable": true, "source": "ecommerce/ecommerce.malloy", "size": "large"}
query: table('malloy-data.ecomm.order_items') -> {
  group_by: status
  aggregate: order_item_count is count()
}
```

The use of `group_by` in the above query invokes a `SELECT` with a `GROUP BY` in SQL. Malloy also has a `project` transformation, which will `SELECT` without a `GROUP BY`.

Notice that after you write this, small "Run | Render" code lens will appear above the query. Run will show the JSON result, while Render will show a table by default, or a visualization if you've configured one (more on this later). Click the code lens to run the query. This will produce the following SQL:

```sql
SELECT
  base.status as status,
  COUNT( 1) as order_item_count
FROM malloy-data.ecomm.order_items as base
GROUP BY 1
ORDER BY 2 desc
```

_Note: To see the SQL being generated by your query, open up a New Terminal in the top menu, then select Output, and pick “Malloy” from the menu on the right._

![Kapture 2021-08-18 at 17 07 03](https://user-images.githubusercontent.com/7178946/130125702-7049299a-fe0f-4f50-aaed-1c9016835da7.gif)


Next question: In 2020, how much did we sell to users in each state? This requires filtering to the year 2020, excluding cancelled and returned orders, as well as joining in the users table.
```malloy
--! {"isRunnable": true, "source": "ecommerce/ecommerce.malloy", "size": "large"}
query: table('malloy-data.ecomm.order_items') {
  join_one: users is table('malloy-data.ecomm.users') on user_id = users.id
} -> {
  where:
    created_at ? @2020,
    status != 'Cancelled' & != 'Returned'
  group_by: users.state
  aggregate: total_sales is sale_price.sum()
}
```

At this point we might notice we’re defining a few things we might like to re-use, so let’s add them to the model:
```malloy
source: users is table('malloy-data.ecomm.users') {
  primary_key: id
}

source: order_items is table('malloy-data.ecomm.order_items') {
  primary_key: id
  join_one: users with user_id
  measure:
    total_sales is sale_price.sum()
    order_count is count(distinct order_id)
}
```

Having defined this in the model, the VSCode plugin will give us handy "Outline" and "Schema" tools in the left menu. "Outline" provides an interactive navigator of the model, in order, and "Schema" shows all of the attributes (both raw fields in the underlying table, and the dimensions, measures, and named queries you've defined in your model).

<img style="width: 281px !important" alt="Screen Shot 2021-08-27 at 4 21 15 PM" src="https://user-images.githubusercontent.com/7178946/131198480-28410cb0-9482-40b4-bc6f-5d67ecdeda2a.png">

Our query is now very simple:
```malloy
--! {"isRunnable": true, "source": "ecommerce/ecommerce.malloy", "size": "large", "queryName": "sales_by_state_2020"}
query: sales_by_state_2020 is order_items {where: created_at ? @2020} -> {
  group_by: users.state
  aggregate: total_sales
}
```

To further simplify, we can add this and a couple other queries we’ll frequently use to our model. Once you define these, the VSCode plugin will supply a “Run” button next to each query:
```malloy
  query: sales_by_state_2020 is {
    where: created_at ? @2020
    group_by: users.state
    aggregate: total_sales
  }

  query: orders_by_status is {
    group_by: status
    aggregate: order_count
  }

  query: sales_by_month_2020 is {
    group_by: order_month is created_at.month
    aggregate: total_sales
  }
```

Allowing us to run the following very simple command next time we want to run any of these queries:
```malloy
--! {"isRunnable": true, "source": "ecommerce/ecommerce.malloy", "size": "large"}
query: order_items -> sales_by_state_2020
```

Note that queries can be filtered at any level. A filter on a source applies to the whole source; one before the fields in a `reduce` or `project` transformation applies to that transformation; and one after an aggregate field applies to that aggregate only. See filters documentation for more information on filter expressions. Here's an example with a variety of filter usage:

```malloy
--! {"isRunnable": true, "source": "ecommerce/ecommerce.malloy", "size": "large"}
query: order_items {
  where:
    users.state = 'California' | 'New York' | 'Texas',
    status != 'Cancelled' & != 'Processing'
} -> {
  group_by: users.state
  aggregate:
    total_sale_price_2020 is sale_price.sum() { where: created_at  ? @2020 },
    percent_items_returned is 100.0 * (count() { where: status = 'Returned' }) / count()
}
```

Queries can contain other nested structures, by including additional transformations as fields, so our named query (`sales_by_month_2020`) can also now be called anywhere as a nested structure. Note that these structures can nest infinitely!:

```malloy
--! {"isRunnable": true, "source": "ecommerce/ecommerce.malloy", "size": "large"}
query: order_items -> {
  group_by: users.state
  aggregate: total_sales
  nest:
    orders_by_status
    sales_by_month_2020
}
```

Which can be visualized using a data_style
```json
{
  "sales_by_month_2020": {
    "renderer": "line_chart"
  },
  "orders_by_status": {
    "renderer": "bar_chart"
  }
}
```
<img width="899" alt="Screen Shot 2021-08-18 at 4 44 01 PM" src="https://user-images.githubusercontent.com/7178946/130128542-e122eab9-5cf2-48cb-b87c-ce520517d595.png">


Putting a few named queries together as nested structures allows us to produce a dashboard with an overview of sales, having written remarkably little code. Use the `dashboard` renderer to format the results like this:

```malloy
--! {"isRunnable": true, "source": "ecommerce/ecommerce.malloy", "size": "large", "queryName": "state_dashboard", "dataStyles": { "sales_by_month_2020": { "renderer" : "line_chart" },  "orders_by_status": { "renderer": "bar_chart" }}}
query: state_dashboard is order_items -> {
  group_by: users.state
  aggregate:
    total_sales
    order_count
  nest:
    orders_by_status
    sales_by_month_2020
}
```