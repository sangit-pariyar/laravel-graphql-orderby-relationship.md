#  Laravel: OrderBy on a Relationship Field (GraphQL, Lighthouse, Scout, Pagination  2025)

> Sort your GraphQL results by related model fields in Laravel with full pagination, Laravel Scout support, and zero raw SQL. Clean, production-grade solution for Laravel 10+ with Lighthouse.

---

## ❓ The Problem

Sorting by relationship fields (e.g., `client.name`, `template.name`) in Eloquent is easy on small apps. But when you're using:

* **GraphQL (Lighthouse)**
* **Laravel Scout for full-text search**
* **Pagination with filters**

... the moment you add `orderBy: { column: "client.name" }`, things break.

No join = no sort. But doing joins with pagination and search gets tricky fast.

---

##  The Goal

Enable GraphQL like this:

```graphql
query {
  listItems(
    companyIdentifier: "abc-123"
    orderBy: {
      column: "client.name"
      direction: ASC
    }
  ) {
    data {
      name
      client {
        clientName
      }
      template {
        name
      }
    }
  }
}
```

That sorts results by `client.clientName`, while still supporting:

* Full-text search
* Pagination
* Clean schema

---

##  1. Define `OrderByInput` in GraphQL

```graphql
input OrderByInput {
  column: String!
  direction: String!
}
```

And add to your resolver query:

```graphql
extend type Query {
  listItems(
    companyIdentifier: ID!
    search: String
    orderBy: OrderByInput
    templateFilter: [String]
    clientFilter: [String]
  ): [Item!]
  @guard
  @canModel(ability: "list", model: "Item", injectArgs: true)
  @paginate(resolver: "App\\GraphQL\\Queries\\ListItems")
}
```

---

##  2. Backend Resolver Logic (Laravel)

In `app/GraphQL/Queries/ListItems.php`:

```php
use App\Models\Item;
use Illuminate\Pagination\LengthAwarePaginator;

final readonly class ListItems
{
    public function __invoke(null $_, array $args): LengthAwarePaginator
    {
        $matchingIds = [];

        if (!empty($args['search'])) {
            $scoutQuery = Item::search($args['search'])
                ->where('companyIdentifier', $args['companyIdentifier']);

            $matchingIds = $scoutQuery->get()->pluck('identifier')->all();

            if (empty($matchingIds)) {
                return new LengthAwarePaginator([], 0, $args['first'], $args['page']);
            }
        }

        $query = Item::query()->where('companyIdentifier', $args['companyIdentifier']);

        if (!empty($matchingIds)) {
            $query->whereIn('identifier', $matchingIds);
        }

        if (!empty($args['templateFilter'])) {
            $query->whereIn('templateIdentifier', $args['templateFilter']);
        }

        if (!empty($args['clientFilter'])) {
            $query->whereIn('clientIdentifier', $args['clientFilter']);
        }

        // Handle ordering
        if (!empty($args['orderBy'])) {
            $column = $args['orderBy']['column'];
            $direction = $args['orderBy']['direction'];

            if ($column === 'client.name') {
                $query->join('clients', 'items.clientIdentifier', '=', 'clients.identifier')
                      ->select('items.*')
                      ->orderBy('clients.clientName', $direction);
            } elseif ($column === 'template.name') {
                $query->join('templates', 'items.templateIdentifier', '=', 'templates.identifier')
                      ->select('items.*')
                      ->orderBy('templates.name', $direction);
            } else {
                $query->orderBy('items.' . $column, $direction);
            }
        }

        $query->orderBy('items.created_at', 'desc');

        $total = $query->count();
        $results = $query->forPage($args['page'], $args['first'])->get();

        return new LengthAwarePaginator($results, $total, $args['first'], $args['page']);
    }
}
```

 Joins are conditional.
 `orderBy` works across relationships.
 No `DB::raw()` or hacks.

---

##  3. Frontend Usage (Vue + Apollo Example)

```js
const columnMap = {
  name: 'name',
  clientName: 'client.name',
  templateName: 'template.name',
};

const queryVars = {
  companyIdentifier: 'abc-123',
  orderBy: {
    column: columnMap[selectedColumn],
    direction: selectedDirection,
  },
  page: currentPage,
  first: perPage,
};
```

```graphql
query ListItems(
  $companyIdentifier: ID!,
  $orderBy: OrderByInput,
  $page: Int!,
  $first: Int!
) {
  listItems(
    companyIdentifier: $companyIdentifier,
    orderBy: $orderBy,
    page: $page,
    first: $first
  ) {
    data {
      name
      client { clientName }
      template { name }
    }
  }
}
```

---

##  Final Results: Relationship Sorting Works!

* ✅ Sort by related model fields (e.g., `client.name`)
* ✅ Works with Laravel Scout
* ✅ Full pagination support
* ✅ GraphQL + Lighthouse compatible
* ✅ No custom directives or raw SQL

---

##  Why This Matters

Modern Laravel apps are full of dashboards and filters. Real users want to sort by things like client names, project titles, template labels — and they’re almost always on related tables.

This solution solves it **cleanly**.

---

##  Author

Written by **Sangit Pariyar**  Full-stack Developer (Laravel · Vue · GraphQL)

* 🔗 [GitHub](https://github.com/sangit-pariyar)
* 🌐 [Portfolio](https://sangit-pariyar.me)

---
