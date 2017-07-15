+++
date = "2017-07-14T21:43:12-04:00"
draft = false
title = "Using Model Factory States To Create Related Models"
description = "Laravel’s model factories can be used to automatically create related models when the factory is called. This is a great way to DRY up the arrange portion of your tests but can cause some problems. In this post we will examine how model factory states can be used to avoid any issues."

+++
Laravel’s model factories can be used to automatically create related models when the factory is called. This is a great way to DRY up the arrange portion of your tests. Consider the following factory definition:

```php
$factory->define(Comment::class, function ($faker) {
	return [
		'post_id' => factory(Post::class)->lazy(),
		'content' => $faker->sentence()
	];
});
```

This factory will create a Comment along with a Post whenever it is called unless the Post is explicitly provided.

```php
// creates a Comment and a Post
factory(Comment::class)->create();

// uses the provided Post instead of creating one
$post = factory(Post::class)->create();
$comment = factory(Comment::class)->create([
	'post_id' => $post->id
]);
```

While the first example can save you from having to repeat the Post creation call across your test suite, I think it suffers from two problems. First, when a Comment is created with  `$comment = factory(Comment::class)->create();` I cannot tell just by looking at the test that a Post has also been created. This implicit behavior could confuse my future self or anyone else working with the code. Second, I will no longer be able to use the factory as an argument to the save() method:

```php
$post->comments()->save(factory(Comment::class)->make());
```

This code will correctly assign the comment to the post, as expected. However, since the post_id is not explicitly provided to the comment factory an additional unassigned Post will be created.

Despite its convenience I think these two problems limit the usefulness of the lazy() method. Fortunately Laravel gives us a solution - model factory states:

```php
$factory->define(Comment::class, function ($faker) {
	return [
		'content' => $faker->sentence()
	];
});

$factory->state(Comment::class, 'withPost', function ($faker) {
    return [
		'post_id' => factory(Post::class)->lazy(),
	];
});
```

By defining a state we can use the base Comment factory as an argument to the save() method without an additional Post being created. When we need a Comment with a Post we can be explicit and use the `withPost` state.

```php
// No unassigned post since the base Comment factory does not define it
$post->comments()->save(factory(Comment::class)->make());

// Creates the Post but is explicit about it
$comment = factory(Comment::class)->states(['withPost'])->create();
```
