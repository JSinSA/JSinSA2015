# StaticElastic: An Elasticsearch and Backbone SPA

* Jurgens du Toit / [@jrgns](http://twitter.com/jrgns)
* [EagerELK](http://blog.eagerelk.com)


# Elasticsearch

![Elasticsearch search trend](http://jrgns.net/talks/static-elastic/img/elasticsearch_trends.png)

* Started out as a search engine / interface to Lucene
* Started evolving into other use cases
* NoSQL?
* Elasticsearch (the company) -> Elastic
* Currently Version 1.6, active development

-----------------------------------------

# Elastic / ELK / Found

* Developed by Elastic (valued at > $700 mil)
* The rest of the ELK stack
* Support Contracts
* Plugins
  - Marvel
  - Shield
  - Watcher
* Talk about the company - community involvement, etc.

-----------------------------------------

### Vitals

![The statistics](http://jrgns.net/talks/static-elastic/img/vitalstatistix.jpg)

* REST interface to a schema-free JSON document store
* Plugin Architecture
* Near Real Time reads
* Search
* Aggregations
* Basic CRUD

-----------------------------------------

### Architecture</h3>

![Architecture](http://jrgns.net/talks/static-elastic/img/architecture.jpeg)

* Runs over Lucene and provides distribution and an API
* Meant to run in a Cluster
* Shards and Replicas provides redundancy
* CAP - Consistency, Availability, Partition tolerance
* Indices and Types
* Mappings

## Backbone

* One of the first MVP libraries / frameworks
* Lightweight
* Easy to parse
* Serves as a prototype of how to implement an SPA using ES as a backend

## SPA: Why Backbone and ES

![Rev it up!](http://jrgns.net/talks/static-elastic/img/fastandfurious.jpg)

* HTTP, REST and CORS out of the box
* Can't serve files like CouchDB, but hey
* Versioning
* Relations
* Routing
* Silos

* Oh, yes, search!
  - highlighting
  - spanning
  - fuzzy search
  - more of the same
  - relational
  - geo

## Why NOT Elasticsearch

![It's gone!](http://jrgns.net/talks/static-elastic/img/needle-in-a-haystack.jpg"><b)

[They're working on it](https://www.elastic.co/guide/en/elasticsearch/resiliency/current/index.html)

* Aphyr June 2014 - 1.1, April 2015 - 1.5
* Sometimes CP, sometimes AP

You can lose documents if

* The network partitions into two intersecting components
* Or into two discrete components
* Or if even a single primary is isolated
* If a primary pauses (e.g. due to disk IO or garbage collection)
* If multiple nodes crash around the same time

* Advice on preventing splitbrain / number of nodes / masters
* Advice on having a master DB replicating into search DB
* Are logs that important?
* Be careful with production data!

* Check the page on their data-loss & consistency issues / fixes. Great praise from Aphyr

* Be sure that you know your types / mappings before going into prod. Otherwise reindex and alias

## StaticElastic

* Load documents
* Search / highlighting
* Fetch document
* Significant terms in the section
* Update
* Update conflict

-----------------------------------------

```javascript
// curl http://localhost:9200/static-elastic/guardian/_search
{
  "took" : 4,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "failed" : 0
  },
  "hits" : {
    "total" : 3938,
    "max_score" : 1.0,
    "hits" : [ {
      "_index" : "static-elastic",
      "_type" : "guardian",
      "_id" : "money-2015-jun-29-digital-banking-mondo-hopes-to-become-the-google-or-facebook-of-the-sector",
      "_score" : 1.0,
      "_source":{"title":"Digital banking:
      //...
```                

-----------------------------------------

```javascript
// curl http://localhost:9200/static-elastic/guardian/business-marketforceslive-2015-jun-29-ocado-jumps-more-than-2-on-hopes-of-international-deal
{
  "_index" : "static-elastic",
  "_type" : "guardian",
  "_id" : "business-marketforceslive-2015-jun-29-ocado-jumps-more-than-2-on-hopes-of-international-deal",
  "_version" : 1,
  "found" : true,
  "fields" : {
    "section" : [ "Business" ],
    "title" : [ "Ocado jumps more than 2% on hopes of international deal" ]
  }
}
```          

-----------------------------------------

```javascript
{
  "took" : 3,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "failed" : 0
  },
  "hits" : {
    "total" : 3938,
    "max_score" : 0.0,
    "hits" : [ ]
  },
  "aggregations" : {
    "keywords" : {
      "buckets" : [ {
        "key" : "Business",
        "doc_count" : 1298
      }, {
        "key" : "Books",
        "doc_count" : 967
      }, {
      //...
```          

-----------------------------------------

```javascript
// Backbone.Model.Page.sync
sync: function(method, model, options) {
  // Setup CORS
  options || (options = {});
  if (!options.crossDomain) { options.crossDomain = true; }

  // Do "partial" updates and fetch the _source field
  if (method === 'update' || method === 'create') {
    options.data = JSON.stringify({ "doc": model });
    options.type = 'POST';
    options.url  = model.url() + '/_' + method + \
      '?fields=_source&version=' + this.attributes._version;
  }

  return Backbone.Model.prototype.sync.call(
    this, method, model, options
  );
}
```          

-----------------------------------------

```javascript
// Backbone.Model.Page.parse (1)
parse: function(result) {
  if (result.get) {
    // This was from a create or an update
    result._source = result.get._source;
    delete result.get;
  }
  // Use the attributes in the `_source` field for the models attribs
  var model = _.extend(
    this.defaults, result._source ? result._source : {}
  );

  // Set the meta fields
  _.each(this.meta_fields, function(field) {
    if (result[field]) { model[field] = result[field]; }
  });
  //...
```          

-----------------------------------------

```javascript
// Backbone.Model.Page.parse (2)
parse: function(result) {
  //...
  // Check the highlighting
  if (result.highlight) {
    model.highlight = result.highlight;
  }
  //...
```    

```html
&lt;!-- Collections Template --&gt;
<% pages.each(function(page) { %>
  <li>
    <a href="/<%= page.attributes._id %>">
      <% if (page.attributes.highlight) { %>
        <%= page.attributes.highlight.title %>
      <% } else { %>
        <%= page.attributes.title %>
      <% } %>
    </a>
  </li>
<% }) %>
```          

-----------------------------------------

```javascript
save: function(key, val, options) {
  if (typeof key === 'object') {
    key._version = this.attributes._version;
  }
  return Backbone.Model.prototype.save.call(this, key, val, options);
},

toJSON: function(options) {
  var data = _.clone(this.attributes);
  // Don't send the meta fields to the server when POST / PUT ing
  _.each(this.meta_fields, function(field) {
    if (data[field]) {
      delete data[field];
    }
  });
  return data;
}
```          

-----------------------------------------

```javascript
// Backbone.View.Pages
loadResults: function(evt) {
  var options = {
    remove: false, processData: this.search === '',
    data: { from: this.from }
  };
  if (this.search !== '') {
    options.data.query = { term: { title: this.search } };
    options.data.highlight = {
      pre_tags: [ '<strong>' ], post_tags: [ '</strong>' ],
      fields: { 'title': {} }
    };
    options.type = 'POST';
    options.data = JSON.stringify(options.data);
  }
  Pages.fetch(options);
},
```          

## Resources

![Resources](http://jrgns.net/talks/static-elastic/img/resources.jpg)
        <aside class="notes" data-markdown>
* Definitive Guide
* Support Contracts / Training
* Discourse
* Meetup - This month - Beginners guide to ES
* How to get started
* Angular Library
* Ansible plays

![Some examples](http://jrgns.net/talks/static-elastic/img/output.gif)

## Questions

## Thanx!

* http://jrgns.net
* http://EagerELK.com
* http://LogstashConfigGuide.com
