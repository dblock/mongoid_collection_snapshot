Mongoid Collection Snapshot
===========================

Easy maintenence of collections of processed data in MongoDB with the Mongoid ODM.

Example:
--------

Suppose that you have a Mongoid model called `Artwork`, stored
in a MongoDB collection called `artworks` and the underlying documents 
look something like:

  { name: 'Flowers', artist: 'Andy Warhol', price: 3000000 }

From time to time, your system runs a map/reduce job to compute the
average price of each artist's works, resulting in a collection called
`artist_average_price` that contains documents that look like:

  { _id: { artist: 'Andy Warhol'}, value: { price: 1500000 } }

If your system wants to maintain and use this average price data, it has 
to do so at the level of raw MongoDB operations, since
map/reduce result documents don't map well to models in Mongoid.
Furthermore, even though map/reduce jobs can take some time to run, you probably 
want the entire `artist_average_price` collection populated atomically
from the point of view of your system, since otherwise you don't ever
know the state of the data in the collection - you could access it in
the middle of a map/reduce and get partial, incorrect results.

mongoid_collection_snapshot solves this problem by providing an atomic
view of collections of data like map/reduce results that live outside
of Mongoid. 

In the example above, we'd set up our average artist price collection like:

    class AverageArtistPrice
      include Mongoid::CollectionSnapshot

      def build
        map = <<-EOS
          function() {
            emit({artist: this['artist']}, {count: 1, sum: this['price']})
          }
        EOS

        reduce = <<-EOS
          function(key, values) {
            var sum = 0;
            var count = 0;
            values.forEach(function(value) {
              sum += value['price'];
              count += value['count'];
            });
            return({count: count, sum: sum});
          }
        EOS

        Artwork.collection.map_reduce(map, reduce, :out => collection_snapshot.name)
      end

      def average_price(artist)
        doc = collection_snapshot.findOne({'_id.artist': artist})
        doc['value']['sum']/doc['value']['count']
      end
    end

Now, if you want
to schedule a recomputation, just call `AverageArtistPrice.create`. The latest
snapshot is always available as `AverageArtistPrice.latest`, so you can write
code like:

    warhol_expected_price = AverageArtistPrice.latest.average_price('Andy Warhol')

And always be sure that you'll never be looking at partial results. The only
thing you need to do to hook into mongoid_collection_snapshot is implement the
method `build`, which populates the collection snapshot and any indexes you need.

By default, mongoid_collection_snapshot maintains the most recent two snapshots 
computed any given time.

