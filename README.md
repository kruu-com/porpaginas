# Porpaginas

This library solves a bunch of issues that comes up with APIs of Repository
classes alot:

- You need different methods for paginatable and non-paginatable finder
  methods.
- You need to expose the underlying data-source and return query objects from
  your repository.

Both Pagerfanta and KnpLabs Pager don't solve this issue and their APIs are
really problematic in this regard. You need to return the query objects or
adapters for paginators from your repositories to get it working and then use
an API on the controller side to turn them into a paginated result set.

Porpaginas solves this by introducing a sane abstraction for paginated results.
For rendering purposes you can integrate with either Pagerfanta or KnpLabs
Pager, this library is not about reimplementating the view part of pagination.

Central part of this library is the interface `Paginatable`:

```php
<?php
namespace Porpaginas;

interface Paginatable extends Countable, IteratorAggregate
{
    /**
     * @param int $offset
     * @return Paginator
     */
    public function take($offset, $limit);

    /**
     * Return the number of all results in the paginatable.

     * @return int
     */
    public function count();

    /**
     * Return an iterator over all results of the paginatable.
     * 
     * @return Iterator
     */
    public function getIterator();
}
```

This API offers you two ways to iterate over the paginatable result,
either the full result or a paginated window of the result using ``take()``.
One drawback is that the query is always lazily executed inside
the ``Paginatable`` and not directly in the repository.

The ``Paginator`` interface returned from ``Paginatable#take()``
looks like this:

```php
<?php

namespace Porpaginas;

interface Paginator extends Countable, IteratorAggregate
{
    /**
     * Return the number of ALL results in the paginatable.

     * @return int
     */
    public function count();

    /**
     * Return an iterator over selected windows of results of the paginatable.
     * 
     * @return Iterator
     */
    public function getIterator();
}
```

## Supported Backends

- Doctrine ORM

## Example

Take the following example using Doctrine ORM:

```php
<?php
class UserRepository extends EntityRepository
{
    /**
     * @return \Porpaginas\Paginatable
     */
    public function findAllUsers()
    {
        $qb = $this->createQueryBuilder('u')->orderBy('u.username');

        return new ORMQueryPagintable($qb);
    }
}

class UserController
{
    /**
     * @var UserRepository
     */
    private $userRepository;

    public function listAction(Request $request)
    {
        $result = $this->userRepository->findAllUsers();
        // no filtering by page, iterate full result

        return array('users' => $result);
    }

    public function pagerfantaListAction(Request $request)
    {
        $result = $this->userRepository->findAllUsers();

        $paginator = new Pagerfanta(new PorpaginasAdapter($result));
        $paginator->setCurrentPage($request->get('page'));

        return array('users' => $paginator);
    }

    public function knplabsListAction(Request $request)
    {
        $result = $this->userRepository->findAllUsers();

        $paginator = $this->get('knp_paginator')->paginate(
            $result,
            $request->get('page'),
            20
        );

        return array('users' => $paginator);
    }
}
```
