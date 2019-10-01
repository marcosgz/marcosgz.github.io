---
layout: post
title:  "Calculate distance between two coordinates using ActiveRecord Arel"
date:   2019-05-24 08:17:15 -0300
categories: ruby-on-rails
tags: arel activerecord coordinates geolocation mysql
---

If you had to calculate the distance between two coordinates you probably found several different ways using the haversine formula:

> The haversine formula determines the [great-circle distance][great-circle-distance] between two points on a [sphere][sphere] given their [longitudes][longitude] and [latitudes][latitude]. Important in navigation, it is a special case of a more general formula in [spherical trigonometry][spherical-trigonometry], the law of haversines, that relates the sides and angles of spherical triangles.

And probably you saw many variations of SQL query like this:

{% highlight sql %}
SELECT
  (
    6371 * acos(
      cos(
        radians(some_latitude)
      ) * cos(
        radians(lat)
      ) * cos(
        radians(lng) - radians(some_longitude)
      ) + sin(
        radians(some_latitude)
      ) * sin(
        radians(lat)
      )
    )
  ) AS distance
FROM locations;
{% endhighlight %}


In practice `some_latitude` and `some_longitude` would be entered in your query as the search parameters and the `lat`/`lng` represents the value already in the database. The average radius of the Earth is 6,371 km (3,960 miles). Multipling the result of the formula by 6371 is used to make sure the results are in kilometers.

If you are using similar SQL multiple times on your code or maybe if you don't like to write raw SQL on your ruby application, here is a small [activerecord][activerecord] and [arel][arel] helper that can be useful to compose SQL query.

{% highlight ruby %}
# frozen_string_literal: true

class ArelSphereDistance
  UNITS = {
    kilometers: 6_371,
    meters: 6_371 * 1_000,
    feets: 3_960 * 5_280,
    miles: 3_960,
  }

  def initialize(lat_from: , lon_from: , lat_to: , lon_to: )
    @lat_from = cast(lat_from)
    @lon_from = cast(lon_from)
    @lat_to = cast(lat_to)
    @lon_to = cast(lon_to)
  end

  def to_arel(unit: :miles)
    ratio = cast(UNITS[unit])
    formula = acos do
      Arel::Nodes::Addition.new(
        Arel::Nodes::Multiplication.new(cos {
          radians { @lat_to }
        }, cos {
          radians { @lat_from }
        }) * cos {
          Arel::Nodes::Subtraction.new(radians { @lon_from }, radians { @lon_to })
        },
        Arel::Nodes::Multiplication.new(
          sin {
            radians { @lat_to }
          },
          sin {
            radians { @lat_from }
          }
        )
      )
    end
    Arel::Nodes::Grouping.new(
      Arel::Nodes::Multiplication.new(ratio, formula)
    ).tap do |instance|
      instance.singleton_class.include Arel::AliasPredication
    end
  end

  protected

  def cast(value)
    return value if value.class.to_s.split('::')[0] == 'Arel'

    Arel::Nodes::SqlLiteral.new(value.to_s)
  end

  def named_function(name, values)
    Arel::Nodes::NamedFunction.new(name, values)
  end

  %w[radians sin cos acos].each do |func|
    define_method func do |&block|
      named_function(func, [block.call])
    end
  end
end

{% endhighlight %}

Here are a few examples of how to use to build your queries:

{% highlight ruby %}
CREATE TABLE `addresses` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `lat` decimal(18,12) DEFAULT NULL,
  `lon` decimal(18,12) DEFAULT NULL,
  PRIMARY KEY (`id`)
);
{% endhighlight %}

{% highlight ruby %}
# frozen_string_literal: true

class Address < ActiveRecord::Base
  validates :lat, presence: true
  validates :lon, presence: true
end
{% endhighlight %}

List the 10 closest addresses of a coordinate with the distance in meters.

{% highlight ruby %}
table = Address.arel_table
distance_between = ArelSphereDistance.new(
  lat_from: table[:lat],
  lon_from: table[:lon],
  lat_to: 40.790505,
  lon_to: -73.971719,
)
Address.select(
  table[Arel.star],
  distance_between.to_arel(unit: :meters).as('distance')
).order('distance asc').limit(10)
# Address Load (1.4ms)  SELECT  `addresses`.*, (6371000 * acos(cos(radians(40.790505)) * cos(radians(`addresses`.`lat`)) * cos(radians(`addresses`.`lon`) - radians(-73.971719)) + sin(radians(40.790505)) * sin(radians(`addresses`.`lat`)))) AS distance FROM `addresses`  ORDER BY distance asc LIMIT 10
{% endhighlight %}

Get addresses that are at least 1km from a given coordinate.

{% highlight ruby %}
Address.where(distance_between.to_arel(unit: :kilometers).lteq(1.0))
# Address Load (14.7ms)  SELECT `addresses`.* FROM `addresses` WHERE ((6371 * acos(cos(radians(40.790505)) * cos(radians(`addresses`.`lat`)) * cos(radians(`addresses`.`lon`) - radians(-73.971719)) + sin(radians(40.790505)) * sin(radians(`addresses`.`lat`)))) <= 1.0)
{% endhighlight %}

MySQL added a few [spatial convenience functions][mysql-spatial-convenience-functions] since version 5.7.6. And one of those convenience functions is `ST_Distance_Sphere`, which allows one to calculate the (spherical) distance between two points. And it makes things much simpler.

PostgreSQL has also a great extension named [PostGIS][postgis] that provides spatial objects. Strongly recommend the use.

If you are looking for more advanced queries or a complete geocode solution take a look at the [geocoder][geocoder] gem. This gem adds several scopes to the [activerecord][activerecord] according to the examples in the documentation.

{% highlight ruby %}
Venue.near('Omaha, NE, US')                   # venues within 20 miles of Omaha
Venue.near([40.71, -100.23], 50)              # venues within 50 miles of a point
Venue.near([40.71, -100.23], 50, units: :km)  # venues within 50 kilometres of a point
Venue.geocoded                                # venues with coordinates
Venue.not_geocoded                            # venues without coordinates
{% endhighlight %}


[great-circle-distance]: https://en.wikipedia.org/wiki/Great-circle_distance
[sphere]: https://en.wikipedia.org/wiki/Sphere
[longitude]: https://en.wikipedia.org/wiki/Longitude
[latitude]: https://en.wikipedia.org/wiki/Latitude
[navigation]: https://en.wikipedia.org/wiki/Navigation
[spherical-trigonometry]: https://en.wikipedia.org/wiki/Spherical_trigonometry
[activerecord]: https://guides.rubyonrails.org/active_record_basics.html
[arel]: https://github.com/rails/rails/tree/master/activerecord/lib/arel
[mysql-spatial-convenience-functions]: https://dev.mysql.com/doc/refman/5.7/en/spatial-convenience-functions.html
[postgis]: https://postgis.net
[geocoder]: https://github.com/alexreisner/geocoder
