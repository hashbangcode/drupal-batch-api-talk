---
theme: uncover
paginate: true
class:
  - lead
  - invert
size: 16:9
style: |
  .small-text {
    font-size: 0.75rem;
  }
  p {
    text-align: left;
  }
  p.centre {
    text-align: center;
  }
footer: "Philip Norton [hashbangcode.com](https://www.hashbangcode.com) [@hashbangcode](https://fosstodon.org/@hashbangcode) [@philipnorton42](https://fosstodon.org/@philipnorton42)"
marp: true
---

# The Drupal Batch API
<p class="centre">Philip Norton<br>
<small>DrupalCamp Scotland 2024</small></p>
<!-- Speaker notes will appear here. -->

---

# Philip Norton
- Developer at Code Enigma
- Writer at `#! code` (www.hashbangcode.com)
- NWDUG host
![bg h:50% right:40%](../src/assets/images/lily58.png)

---

## Source Code
- This presentation is available at
<small>https://github.com/hashbangcode/drupal-batch-api-talk</small>
- All code seen in this presentaiton is available at
<small>https://github.com/hashbangcode/drupal_batch_examples</small>
- I have also written extensively about the Batch API on <small>https://www.hashbangcode.com/</small>

---

# The Drupal Batch API

<!--
The Drupal Batch API is a system that allows us to split up large tasks into smaller chunks so that we don't overload the web server. This is a really important concept to understand if you are developing your own Drupal modules, especially if you want to allow users to perform actions on lots of items at once.
In this session we will look at why we need the Drupal Batch API and how it prevents issues. We'll then do a dive into the code surrounding the Batch API and how to run your own batch operations.
There will be a few demos on running the Batch API to perform some simple tasks.
Finally, we'll look at how Drupal uses the Batch API internally and some use cases of the Batch API in contributed modules.
-->
---

## The Batch API
Allows data to be processed in small chunks in order to prevent timeout errors or memory problems.

---

# What Problem Are We Solving?

---

## Bouncing Users
- Users get bored quickly.
- Studies show that a 5 second page load has a 0.6% conversion rate.
- Reducing this to 2 seconds doubes the conversion rate.
- This still means that after 2 seconds 98% of users will assume the page will not do anything.


<!--
Source: https://www.cloudflare.com/learning/performance/more/website-performance-conversion-rates/
-->

---

## Server Timeouts

- Servers are designed to throw errors is something takes too long. Some defaults:

  - PHP (`max_execution_time`) - 30 seconds
  - PHP (`memory_limit`) - 256MB (recommended for Drupal)
  - Apache (`TimeOut`) - 60 seconds
  - Nginx (`send_timeout`/`proxy_send_timeout`) - 60 seconds

<!--
- You can override all these options.
- But, setting them high would cause problems on your web server.
- Think about how long a page request should take in you web server.
2-5 seconds MAX!
- The more resources you give to your web server, the users you can accommodate at once.
-->

---

## The Problem

- Trying to do too much in one page request.
  - Downloading lots of data from an api.
  - Create/update/delete lots of entities.
<br>
- Users assume page is broken and click away.
- The page times out or runs out of memory.

---
<!-- _footer: "" -->
## The Batch API

- Solves these problems by splitting long tasks into smaller chunks.
- Drupal then runs them through a special interface.

![left:center](../src/assets/images/batch_process_running.png)

---

# The Batch API

---

## The Batch Process

The Batch API can be thought of as the following:

- <strong>Initialise</strong> - Set up the batch run, define callbacks.
- <strong>Process</strong> - The batch process operations.
- <strong>Finish</strong> - A finish callback.

---

## Initialise

The BatchBuilder class is used to setup the batch.

```php
use Drupal\Core\Batch\BatchBuilder;
$batch = new BatchBuilder();
```

---
## Initialise

A number of methods set up different parameters.

```php
$batch = new BatchBuilder();
$batch->setTitle('Running batch process.')
  ->setFinishCallback([self::class, 'batchFinished'])
  ->setInitMessage('Commencing')
  ->setProgressMessage('Processing...')
  ->setErrorMessage('An error occurred during processing.');
```

---
## Initialise - Adding Operations

Populate the operations we want to perform.

```php
// Create 10 chunks of 100 items.
$chunks = array_chunk(range(1, 1000), 100);

// Process each chunk in the array.
foreach ($chunks as $id => $chunk) {
  $args = [
    $id,
    $chunk,
  ];
  $batch->addOperation([BatchClass::class, 'batchProcess'], $args);
}
```

---

## Process

Set the batch running by calling `toArray()` and passing the array to `batch_set()`.

```php
batch_set($batch->toArray());
```

The whole purpose of `BatchBuilder` is to generate that array.
This will trigger and start up the batch process.
<!--
- This will redirect to the batch processor interface and call the process operations.
- Yes you can also just define the array and send it to batch_set(), but I find BatchBuilder a nicer interface.
-->

---
<!-- _footer: "" -->
## Process

- The callbacks defined in the `addOperation()` method.
- Parameters are the array of arguments you set.
- `$context` is passed as the last parameter is used to track progress.

```php
public static function batchProcess(int $batchId, array $chunk, array &$context): void {
}
```

- This method is called multiple times (depending on the batch run).

---
<!-- _footer: "" -->
## Process - Tracking Progress

- The `$context` parameter is an array that is maintained between different batch calls.
- The `"sandbox"` element is used inside the batch process and is deleted at the end of the batch run.
- The `"results"` element is will be passed to the finished callback and is often used to track progres for reporting.

```php
public static function batchProcess(int $batchId, array $chunk, array &$context): void {
    if (!isset($context['sandbox']['progress'])) { }
    if (!isset($context['results']['updated'])) { }
}
```

---

## Process - Messages

- As the batch runs you can set a `"message"` element to print messages to the user.
- This will appaer above the batch progress bar as the batch progresses.

```php
// Message above progress bar.
$context['message'] = t('Processing batch #@batch_id batch size @batch_size for total @count items.', [
  '@batch_id' => number_format($batchId),
  '@batch_size' => number_format(count($chunk)),
  '@count' => number_format($context['sandbox']['max']),
]);
```

---

## Finish - The Finished Callback

- When the batch finishes the finished callback is triggered.
- This has a set of parameters that detail how the batch performed.

```php
public static function batchFinished(
  bool $success, 
  array $results,
  array $operations,
  string $elapsed): void {
}
```
<!--
- $success - TRUE if all batch API tasks were completed successfully.
- $results - An results array from the batch processing operations.
- $operations - A list of the operations that had not been completed.
- $elapsed - Batch.inc kindly provides the elapsed processing time in seconds.
-->
---

## Finished - The Finished Callback

For example, you might want to report the results of the batch run to your user.

```php
  public static function batchFinished(bool $success, array $results, array $operations, string $elapsed): void {
    $messenger = \Drupal::messenger();
    if ($success) {
      $messenger->addMessage(t('@process processed @count, skipped @skipped, updated @updated, failed @failed in @elapsed.', [
        '@process' => $results['process'],
        '@count' => $results['progress'],
        '@skipped' => $results['skipped'],
        '@updated' => $results['updated'],
        '@failed' => $results['failed'],
        '@elapsed' => $elapsed,
      ]));
    }
}
```

---

<!-- _footer: "" -->
## The Running Batch

![width:30cm center](../src/assets/images/batch_process_running_requests.png)

<!-- 
- The batch JavaScript called the /batch endpoint.
- This handles the processing of the batch operations.
-->

---

# The Batch "finished" State

---

## The Batch "finished" State

- So far, we have looked at pre-confgured batch runs.
- A better approach is to use the `finished` property of the batch `$context` array.
- If we set this value to >= 1 then the batch process is considered finished.

```php
if (done) {
  $context['finished'] = 1;
}
```

---

## The Batch "finished" State

The setup is slightly different as we only create a single operation.

```php
$array = range(1, 1000);
$batch->addOperation([BatchClass::class, 'batchProcess'], [$array]);
```

This is run over and over until we issue the finished state.

---

## The Batch "finished" State

It is common to divide the progress by the maximum number of items.

```php
$context['finished'] = $context['sandbox']['progress'] / $context['sandbox']['max'];
```

---
<!-- _footer: "" -->
## The Batch "finished" State

This also means that we can just launch the batch with no arguments.

```php
$batch->addOperation([BatchProcessNodes::class, 'batchProcess']);
```

The `max` property is found in the batchProcess() method the first time it is run.

```php
public static function batchProcess(array &$context): void {
  if (!isset($context['sandbox']['progress'])) {
    $query = \Drupal::entityQuery('node');
    $query->accessCheck(FALSE);
    $context['sandbox']['progress'] = 0;
    $context['sandbox']['max'] = $query->count()->execute();
  }
```

---

## Batch Internal Workings

- The Batch API is really an extension of the Queue system.
- When you add operations to the batch you are adding items to the queue.
- The Drupal batch runner then pulls items out of the queue and feeds them to the process method.

---

## When To Use The Batch API

- If the request processes lots of items them move it into a batch.
- Use the batch system early to save having to rework things later.

---

# Running Batch With Drush

---
## Drush

Call batch set as normal.

```php
batch_set($batch->toArray());
```
Then call the Drush function.
```php
drush_backend_batch_process();
```
This will run the batch on the command line. 

---

## Drush

- Be careful! Drush will process the batch operations in the same memory space.
- As you are on the command line you won't time out, but you can run out of memory.

---

# Examples Of Batch API In Action

Some live demos!

---

## Batch Using A Form
- Look at 1,000 items and roll a dice.

---

## Batch Using Drush
- Look at 1,000 items and roll a dice.

---

## Process a CSV file
- Import 1,000 nodes using a batch process.

---

# The Batch API Inside Drupal

---

## The Update Hook

- Update hooks get a `$sandbox` variable. This is actually a batch `$context` array.
- You can set the `#finished` property in the `$sandbox` array to stop the batch.

---
<!-- _footer: "" -->
## The Update Hook
An example of a batched update hook.
```php
function batch_update_example_update_10001(&$sandbox) {
  if (!isset($sandbox['progress'])) {
    $sandbox['progress'] = 0;
    $sandbox['max'] = 1000;
  }

  for ($i = 0; $i < 100; $i++) {
    // Keep track of progress.
    $sandbox['progress']++;
    // Do some actions...
  }
  \Drupal::messenger()->addMessage($sandbox['progress'] . ' items processed.');

  $sandbox['#finished'] = $sandbox['progress'] / $sandbox['max'];
}
```

---

## General Batch Uses

- Drupal also makes use of the Batch API in lots of different situations. For example:
  - Installing Drupal.
  - Deleting users.
  - Bulk content updates.
  - Installing modules.
  - Importing translations.
  - Importing configuration.
  - And much more!

---
<!-- _footer: "" -->

## Top Tips

- If the data needs to be processed in real time then use a batch; otherwise use a standard queue.
- Kick off your batches in a form or controller, but process the batch in a separate class. This allows easy Drush integration.
- Use the `finished` property to make dynamic batches; rather than preloaded.
- Keep your batch operations simple. Breat them apart into separate operations if needed.
- Allow batch operations to pick up where they left off.

<!--
- Think about the footprint of your batch operations. Keep them small.
-->
---

# Modules That Use Batch

---

## View Batch Operation

- Batch process items in a view.

<small>https://www.drupal.org/project/views_bulk_operations</small>

---

## Advanced Queue

- Shows a breakdown of the current queues in your system.
- Gives the option to process queues as a batch run.

<small>https://www.drupal.org/project/advancedqueue</small>

---

## Views Data Export

- A Views plugin that exports data in a number of different formats.

<small>https://www.drupal.org/project/views_data_export</small>

---

## Batch Plugin

- Wraps the Batch API in a plugin to make your batch operations pluggable.

<small>https://www.drupal.org/project/batch_plugin</small>

---

## Resources

- [Drupal 11: An Introduction To Batch Processing With The Batch API](https://www.hashbangcode.com/article/drupal-11-introduction-batch-processing-batch-api)
- [Drupal 11: Batch Processing Using Drush](https://www.hashbangcode.com/article/drupal-11-batch-processing-using-drush)
- [Drupal 11: Using The Finished State In Batch Processing](https://www.hashbangcode.com/article/drupal-11-using-finished-state-batch-processing)
- [Drupal 11: Using The Batch API To Process CSV Files](https://www.hashbangcode.com/article/drupal-11-using-batch-api-process-csv-files)
- [Drupal Batch Examples source code](https://github.com/hashbangcode/drupal_batch_examples/)

---

## Questions?

- Slides: https://github.com/hashbangcode/drupal-batch-api-talk

![bg h:50% right:40%](../src/assets/images/qr_slides.png)

---

## Thanks!

- Slides: https://github.com/hashbangcode/drupal-batch-api-talk

![bg h:50% right:40%](../src/assets/images/qr_slides.png)
