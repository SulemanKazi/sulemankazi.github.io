## Testing a claim about the muslim call to prayer with open data

* Table of Contents
{:toc}

The original purpose of this writeup was to create a visualization for how the Azan (The Muslim call to the five daily prayers, see details below) moves across the globe in a period of 24-hours. However after finishing the visualization, I remembered that I read an article about a decade ago which claimed that there is no time during the year when the Azan is not happening somewhere on the globe. This visualization I made is in the video below and shows how the Azan moves across the world during a 24H period for a given date:

<p align="center"><iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/IQg3wbQmg7U" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe></p>
<p align="center"> </p>

Trying to search for any evidence for this claim led to an unsubstantiated, [vague](https://www.quora.com/Is-the-sound-of-the-Islamic-call-to-prayer-azan-non-stop-across-the-globe) description and some questionable [animations](https://www.youtube.com/watch?v=Q2Rsq6UmfLc) with no data to back them up. I am curious to know if the claim is true. Let's walk through how we can use python, freely available open datasets and some heuristics to prove/disprove this and see what we find! All source code is open and reproducible.

## Azan Basics

Muslims pray five times a day, each prayer is to be performed during a specific interval of the day, determined by the position of the sun. The timings are not fixed (wrt the 24 hour clock) and change as the length of the day and night varies throughout the year. The Azan occurs at the start of each interval for the five prayers (Fajr, Dhuhr, Asr, Maghrib and Isha).

There are slight [differences](https://www.calislamic.com/fifteen-or-eighteen-degrees-calculating-prayer-and-fasting-times-in-islam/) in the way the solar angles are calculated for Fajr/Isha prayers, in addition there are [edge cases](https://www.astronomycenter.net/accut.html#alt) at higher latitudes (for example the sun never sets in some places during the year) which have to be treated differently.

## The Process

In order to determine if the original claim is true we have to:

1. Determine **locations** in the world where the Azan occurs.
2. Determine **Azan times** at all locations from step 1 for every day of the year.
3. Determine if there is any time during the entire year where **none** of the locations above have the Azan happening.

We will use some assumptions during this process, which we'll walk through below. All assumptions made are preceded by the _[Assumption]_  tag.

### 1. Determine locations where the Azan occurs

Ok first things first, we can't expect the Azan to occur at every single point on the globe. The Azan is given whenever muslims pray **in congregation**. _[Assumption]_  A good proxy for prayer in congregation (and hence a place where the Azan occurs) is a mosque, although muslims frequently pray in congregation outside of mosques as well. Ok great, so we need a way to determine where all mosques are in the world! Such a thing unfortunately does not exist. However a [Deloitte study](https://www2.deloitte.com/xe/en/pages/financial-services/articles/the-digital-islamic-services-landscape.html) estimated there were 3.6 million mosques around the world, projected to reach 3.85 million by 2019. Thus when the study was conducted there was 1 mosque for every 500 muslims in the world.

The [GeoNames](https://www.geonames.org/) geographical database provides a list of cities around the globe with a population of >1000, along with their lat/longs and estimated population. Here's what it looks like:

![GeoNames Cities](/imgs/azan/geonames_df.png)

PEW has a [dataset](https://www.pewforum.org/2015/04/02/religious-projection-table/2020/percent/all/) of religious composition of countries around the world.

![PEW Composition](/imgs/azan/pew_df.png)

We can combine this with the GeoNames dataset. i.e. multiply the city population from GeoNames with the muslim percentage in the country the city is in to find the estimated number of muslims in each city. _[Assumption]_ We're ignoring all rural (non-city) areas and assuming that the PEW country-wide percentages are applicable to cities. Since this assumption is in all likelihood, not valid, we'll compensate for it by only considering cities with >1000 muslims, double the 1:500 mosque:muslim ratio we saw above (and Assuming such cities will have a mosque or a prayer in congregation). 

Ok, now we need to verify that our assumptions make sense. Some Googe-Maps searches for "mosque" in a randomly selected sample of the cities we find using the criteria above shows that this assumption is reasonable, and most of them do indeed have mosques. The only exception I found to this rule was in places where the PEW survey had estimated a Muslim population of 1%. I assume this happened because PEW rounds up all population numbers in the range [0 - 1]% to 1%, so we'll special case such countries, and our final heuristic for filtering cities will be:

* If a country has a >1% Muslim population:
  * Only include cities where the estimated muslim population in the city >1000
* If a country has a <=1% Muslim population:
  * Only include **big** cities (population > 1 million).
* If a city's population is unavailable (which is the case for some cities in the GeoNames dataset):
  * Only include cities in countries with a muslim population >50%

Our final heuristic is tuned towards **precision** rather than **recall**, since the criteria for inclusion is strict and we're less concerned with missing some cities with mosques/muslims-praying-in-congregation, and more concerned about our heuristic being strict enough to not over-count. The original dataset has more ~1.2 Million cities, after applying our filter, we're left with just **23006**. These cities are mapped using kepler.gl in the image below, the colour represents the country they are located in, the size is weighted by population:

![A map of filtered cities](/imgs/azan/all_cities.png)

### 2. Determine Azan times at all locations

As we saw above there are a few different ways and edge-cases in calculating Azan times. For the purposes of this experiment, we'll use the python version of the [PrayTimes](http://praytimes.org/) library, which includes different methods of calculating Azan times as well as high latitude corrections built-in. We'll iterate over all cities in our filtered list, and calculate the Azan times for each day of the year. For the purposes of the calculation we'll use the _University of Islamic Sciences, Karachi_ method which sets the angle for both Fajr and Isha prayers at 18°. This is a simplifying _[Assumption]_ because different mosques will use different (equally valid) methods.

Since the PrayTime library also provides sunrise/sunset times, we're able to sanity check the output by selecting a few random locations and ensuring that the times predicted by the library are within a minute of publicly available sunrise/sunset times. I also cross-checked Azan times at several cities with publicly available prayer times and found them to be accurate.

![Sunrise/Sunset Verification](/imgs/azan/sunset_tokyo.png)
_This image shows the sunset time from PrayerTime matches the sunset time from Google to the exact minute (The time on the map is GMT)_


### 3. Visualizing Azan times

Let's visualize what we have so far, since it makes for a nice way of conceptualizing how prayer times move around the world. We'll do this for one day of the year. I discovered [kepler.gl](http://kepler.gl/) (thanks Uber!) which makes it super-simple to visualize geo-data and supports visualization from a CSV file which makes life easy since we've been using Pandas dataframes for our work. Here's what the dataframe that we will export to Kepler looks like:

![Kepler DF](/imgs/azan/kepler_df.png)

After uploading the data, this is what we end up with (for a given time range):

![Kepler Visualization](/imgs/azan/kepler_vis.png)

Each dot represents one of our filtered cities at the time (within our range) when Azan occurs there, the color represents which of the five prayers the Azan is for. The video at the top of this article shows this animated for one day. Someone with more webdev experience can probably make this into a live-view thing where there's a live map hosted showing you where in the world the Azan is occuring in realtime.

### 4. Check if the Azan is continuously occuring

Let's generate the azan time data for each city by looping over each day in the eyar. For each day we will check if for **each minute** of the day, there is atleast **1** city in our list where we have the Azan. Technically since the Azan is about 2-3 minutes long, we can get away with checking every other minute. However remember we said we're focusing on precision, not recall so we'll just check every minute. After stuffing our time-check into a loopwe find that there is **no minute** where the Azan isn't occurring somewhere on Earth. There are 11 (non-contiguous) minutes in the whole year where it occurs at only three cities, but I have manually verified the presence of a mosque **in** or **near** those cities.

There we have it! An answer to a question I had a decade ago. Keeping in mind we made some assumptions this of course isn't a **definitive** answer, but it's a good enough approximation for me to be confident that the results are valid!

## Source Code

All source code and links to datasets are available in this [github repository](https://github.com/SulemanKazi/AzanVis).

## References

1. [GeoNames cities dataset.](https://public.opendatasoft.com/explore/dataset/geonames-all-cities-with-a-population-1000/table/?disjunctive.cou_name_en&sort=name)
2. [PEW Religious Composition.](https://www.pewforum.org/2015/04/02/religious-projection-table/)
3. [PrayTimes library.](http://praytimes.org/)

