# Handling Relations

API Platform Admin handles `to-one` and `to-many` relations automatically.

Thanks to [the Schema.org support](schema.org.md), you can easily display the name of a related resource instead of its IRI.

## Embedded Relations

If a relation is an array of [embeddeds or an embedded](../core/serialization.md#embedding-relations) resource,
by default, the admin automatically replaces the embedded resources' data by their IRI.
However, the embedded data is inserted to a local cache: it will not be necessary to make more requests if you reference some fields of the embedded resource later on.

If you need to edit the embedded data or if you want to display some of the nested fields by using the dot notation for complex structures,
you can keep the embedded data by setting the `useEmbedded` parameter of the Hydra data provider to `true`.

```javascript
// admin/src/App.js

import { HydraAdmin, fetchHydra, hydraDataProvider } from "@api-platform/admin";
import { parseHydraDocumentation } from "@api-platform/api-doc-parser";

const entrypoint = process.env.REACT_APP_API_ENTRYPOINT;

const dataProvider = hydraDataProvider({
    entrypoint,
    httpClient: fetchHydra,
    apiDocumentationParser: parseHydraDocumentation,
    mercure: true,
    useEmbedded: true,
});

export default () => (
    <HydraAdmin
        dataProvider={dataProvider}
        entrypoint={entrypoint}
    />
);
```

The embedded data will be displayed as text field and editable as text input: the admin cannot determine the fields present in it.
To display the fields you want, see [this section](handling-relations.md#display-a-field-of-an-embedded-relation).

This behavior will be the default one in 3.0.

## Display a Field of an Embedded Relation

If you have an [embedded relation](../core/serialization.md#embedding-relations) and need to display a nested field, the code you need to write depends of the value of `useEmbedded` of the Hydra data provider.

If `false` (default behavior), make sure you write the code as if the relation needs to be fetched as a reference.

In this case, you *cannot* use the dot separator to do so.

Note that you cannot edit the embedded data directly with this behavior.

For instance, if your API returns:

```json
{
  "@context": "/contexts/Book",
  "@id": "/books",
  "@type": "hydra:Collection",
  "hydra:member": [
    {
      "@id": "/books/07b90597-542e-480b-a6bf-5db223c761aa",
      "@type": "http://schema.org/Book",
      "title": "War and Peace",
      "author": {
        "@id": "/authors/d7a133c1-689f-4083-8cfc-afa6d867f37d",
        "@type": "http://schema.org/Author",
        "firstName": "Leo",
        "lastName": "Tolstoi"
      }
    }
  ],
  "hydra:totalItems": 1
}
```

If you want to display the author first name in the list, you need to write the following code:

```javascript
import {
  HydraAdmin,
  FieldGuesser,
  ListGuesser,
  ResourceGuesser
} from "@api-platform/admin";
import { ReferenceField, TextField } from "react-admin";

const BooksList = (props) => (
  <ListGuesser {...props}>
    <FieldGuesser source="title" />
    {/* Use react-admin components directly when you want complex fields. */}
    <ReferenceField label="Author first name" source="author" reference="authors">
      <TextField source="firstName" />
    </ReferenceField>
  </ListGuesser>
);

export default () => (
  <HydraAdmin entrypoint={process.env.REACT_APP_API_ENTRYPOINT}>
    <ResourceGuesser
      name="books"
      list={BooksList}
    />
    <ResourceGuesser name="authors" />
  </HydraAdmin>
);
```

If the `useEmbedded` parameter is set to `true` (will be the default behavior in 3.0), you need to use the dot notation to display a field:

```javascript
import {
  HydraAdmin,
  FieldGuesser,
  ListGuesser,
  ResourceGuesser
} from "@api-platform/admin";
import { TextField } from "react-admin";

const BooksList = (props) => (
  <ListGuesser {...props}>
    <FieldGuesser source="title" />
    {/* Use react-admin components directly when you want complex fields. */}
    <TextField label="Author first name" source="author.firstName" />
  </ListGuesser>
);

export default () => (
  <HydraAdmin entrypoint={process.env.REACT_APP_API_ENTRYPOINT}>
    <ResourceGuesser
      name="books"
      list={BooksList}
    />
  </HydraAdmin>
);
```

## Using an Autocomplete Input for Relations

Let's go one step further thanks to the [customization capabilities](customizing.md) of API Platform Admin by adding autocompletion support to form inputs for relations.

Let's consider an API exposing `Review` and `Book` resources linked by a `many-to-one` relation (through the `book` property).

This API uses the following PHP code:

```php
<?php
// api/src/Entity/Review.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiResource;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
#[ApiResource]
class Review
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column(type: 'integer')]
    public ?int $id = null;

    #[ORM\ManyToOne]
    public Book $book;
}
```

```php
<?php
// api/src/Entity/Book.php

namespace App\Entity;

use ApiPlatform\Core\Annotation\ApiFilter;
use ApiPlatform\Core\Annotation\ApiResource;
use ApiPlatform\Core\Bridge\Doctrine\Orm\Filter\SearchFilter;
use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
#[ApiResource]
class Book
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column(type: 'integer')]
    public ?int $id = null;

    #[ORM\Column]
    #[ApiFilter(SearchFilter::class, strategy: 'ipartial')]
    public string $title;

    #[ORM\OneToMany(targetEntity: Review::class)]
    public Collection $reviews;

    public function __construct()
    {
        $this->reviews = new ArrayCollection();
    }
}
```

Notice the "partial search" [filter](../core/filters.md) on the `title` property of the `Book` resource class.

Now, let's configure API Platform Admin to enable autocompletion for the relation selector:

```javascript
import {
  HydraAdmin,
  ResourceGuesser,
  CreateGuesser,
  EditGuesser,
  InputGuesser
} from "@api-platform/admin";
import { ReferenceInput, AutocompleteInput } from "react-admin";

const ReviewsCreate = props => (
  <CreateGuesser {...props}>
    <InputGuesser source="author" />
    <ReferenceInput
      source="book"
      reference="books"
      label="Books"
      filterToQuery={searchText => ({ title: searchText })}
    >
      <AutocompleteInput optionText="title" />
    </ReferenceInput>

    <InputGuesser source="rating" />
    <InputGuesser source="body" />
    <InputGuesser source="publicationDate" />
  </CreateGuesser>
);

const ReviewsEdit = props => (
  <EditGuesser {...props}>
    <InputGuesser source="author" />

    <ReferenceInput
      source="book"
      reference="books"
      label="Books"
      filterToQuery={searchText => ({ title: searchText })}
    >
      <AutocompleteInput optionText="title" />
    </ReferenceInput>

    <InputGuesser source="rating" />
    <InputGuesser source="body" />
    <InputGuesser source="publicationDate" />
  </EditGuesser>
);

export default () => (
  <HydraAdmin entrypoint={process.env.REACT_APP_API_ENTRYPOINT}>
    <ResourceGuesser
      name="reviews"
      create={ReviewsCreate}
      edit={ReviewsEdit}
    />
  </HydraAdmin>
);
```

If the book is embedded into a review and if the `useEmbedded` parameter is set to `true` (will be the default behavior in 3.0),
you need to change the `ReferenceInput` for the edit component:

```javascript
import {
  HydraAdmin,
  ResourceGuesser,
  CreateGuesser,
  EditGuesser,
  InputGuesser
} from "@api-platform/admin";
import { ReferenceInput, AutocompleteInput } from "react-admin";

const ReviewsCreate = props => (
  <CreateGuesser {...props}>
    <InputGuesser source="author" />
    <ReferenceInput
      source="book"
      reference="books"
      label="Books"
      filterToQuery={searchText => ({ title: searchText })}
    >
      <AutocompleteInput optionText="title" />
    </ReferenceInput>

    <InputGuesser source="rating" />
    <InputGuesser source="body" />
    <InputGuesser source="publicationDate" />
  </CreateGuesser>
);

const ReviewsEdit = props => (
  <EditGuesser {...props}>
    <InputGuesser source="author" />

    <ReferenceInput
      source="book"
      reference="books"
      label="Books"
      filterToQuery={searchText => ({ title: searchText })}
      format={v => v['@id'] || v}
    >
      <AutocompleteInput optionText="title" />
    </ReferenceInput>

    <InputGuesser source="rating" />
    <InputGuesser source="body" />
    <InputGuesser source="publicationDate" />
  </EditGuesser>
);

export default () => (
  <HydraAdmin entrypoint={process.env.REACT_APP_API_ENTRYPOINT}>
    <ResourceGuesser
      name="reviews"
      create={ReviewsCreate}
      edit={ReviewsEdit}
    />
  </HydraAdmin>
);
```

The autocomplete field should now work properly!
