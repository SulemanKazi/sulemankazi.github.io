## Investigating a claim about the Muslim call to prayer with Python and Open Data

<iframe src="https://giphy.com/embed/sFgvfo0O6S4iYLl0uB" width="800" height="457" frameBorder="0" class="giphy-embed" allowFullScreen></iframe><p></p>

* Table of Contents
{:toc}

I read an article about a decade ago which claimed that there is no time on Earth that the Azan (The Muslim call to the five daily prayers, see details below) is not happening somewhere around the globe.

Trying to search for any evidence for this claim led to the same unsubstantiated [vague](https://www.quora.com/Is-the-sound-of-the-Islamic-call-to-prayer-azan-non-stop-across-the-globe)description and some questionable [animations](https://www.youtube.com/watch?v=Q2Rsq6UmfLc) with no data to back them up. I am curious to know if the claim is true. Let&#39;s walk through how we can use python, freely available datasets and some heuristics to prove/disprove this and see what we find!

## Azan Basics

Muslims pray five times a day, each prayer is to be performed during a specific interval of the day, determined by the position of the sun. The timings are not fixed (wrt the 24 hour clock) and move as the length of the day and night change throughout the year. The Azan occurs at the start of each interval for the five prayers.

There are slight [differences](https://www.calislamic.com/fifteen-or-eighteen-degrees-calculating-prayer-and-fasting-times-in-islam/) in the way the solar angles are calculated for Fajr/Isha prayers, in addition there are [edge cases](https://www.astronomycenter.net/accut.html#alt) at higher latitudes which have to be treated differently.

## The Process

In order to determine if the original claim is true we have to:

- Determine **locations** in the world where the Azan occurs (i.e. wherever muslims pray in congregation).
- Determine **Azan times** at alllocations from step 1 for 365 days of the year.
- Determine if there is any minute in the 525600 minutes of the year where **none** of the locations above have the Azan happening.

We will use some approximations for all of the steps above, which we&#39;ll walk through below. All assumptions made are preceded by the _[Assumption]_ tag.

## Where does the Azan occur?

Ok first things first, we can&#39;t expect the Azan to occur at every single point on the globe. The Azan is given whenever muslims pray **in**** congregation**. A good proxy for prayer in congregation is a mosque _[Assumption]_ (although muslims frequently pray in congregation outside of mosques as well). Ok great, so we need a dataset of all mosques in the world! Such a thing unfortunately does not exist. However a [Deloitte study](https://www2.deloitte.com/xe/en/pages/financial-services/articles/the-digital-islamic-services-landscape.html) estimates there are 3.6 million mosques around the world (Expected to reach 3.85 million by 2019). Thus when the study was conducted there was 1 mosque for every 500 muslims in the world.

The [GeoNames](https://www.geonames.org/) geographical database provides a list of cities around the globe with a population of \&gt;1000, along with their lat/longs and estimated population. Combining this with PEW&#39;s [estimate](https://www.pewforum.org/2015/04/02/religious-projection-table/2020/percent/all/) of religious composition a the country level, we can multiply the city population with the muslim %age to find the estimated number of muslims in each city _[Assumption]_ Here we&#39;re ignoring all rural (non-city) areas and assuming that the PEW country-based %age of muslims is applicable to all cities in the country. We&#39;re also assuming every city \&gt; 1000 muslims will have a mosque (or a prayer in congregation). Some Googe-Maps searches for mosques in a randomly selected sample of such citeis shows that this assumption is reasonable, and most of them do indeed have mosques. The only exception I found to this rule was in places where the PEW survey had estimated a Muslim population of 1%. I assume this happened because PEW rounds up all population numbers in the range [0 - 1]% to 1%.

So for our final heuristic for filtering cities the criteria we use is:

1. If a country has a \&gt;1% Muslim population:
  1. Only include cities where the estimated muslim population in the city \&gt; 1000
2. If acountry has a \&lt; 1% Muslim population:
  1. Only include **big** cities, i.e. those which have a population of more than 1M
3. If a cities population is unavailable (which is the case for some cities in the GeoNames dataset):
  1. Only include cities in countries with a muslim population \&gt;50%

This heuristic is tuned towards **precision** rather than **recall** , we&#39;re less concerned with missing some cities with mosques, and more concerned about saying a city has a mosque if it doesn&#39;t.

## Determine Azan Times

As we saw above there are a few different ways and edge-cases in calculating Azan times, for the purposes of this experiment, we&#39;ll use the python version of the [PrayTimes](http://praytimes.org/) library, which includes different methods of calculating Azan times as well as high latitude correction. We&#39;ll iterate over all cities in our filtered list, and calculate the Azan times for each day of the year. For the purposes of the calculation we&#39;ll use the _University of Islamic Sciences, Karachi_ method which sets the angle for both Fajr and Isha prayers at 18°. This is an _[Assumption]_ because different mosques will use different (equally valid) methods.

Since the PrayTime library also provides sunrise/sunset times, we&#39;re able to sanity check the output by selecting a few random cities and ensuring that the times predicted by the library are within a minute of publicly available sunrise/sunset times. I also cross-checked prayertimes at several cities with publicly available prayer times and found them to be accurate.

For each day we will check if for **each minute** of the day, there is atleast **1** city in our list where we have Azan time. Technically since on average the Azan is about 2-3 minutes long, we can get away with checking every other minute, but since we&#39;re focusing on precision, not recall we&#39;ll just check every minute.

## Visualizing the Azan

Let&#39;s visualize what we have so far, i.e. Azan locations around the globe. We&#39;ll do this for one day of the year. I discovered [kepler.gl](http://kepler.gl/) which makes it super-simple to visualize geo-data and supports visualization from a CSV file which makes life easy since we&#39;ve been using Pandas dataframes for our work. After uploading the data, this is what we end up with, each dot represents one of our filtered cities at the time when Azan occurs there, the color represents which of the five prayers the Azan is for (someone with more webdev experience can probably make this into a live-view thing where there&#39;s a live map showing you where the Azan is occurring on the globe at the current time).

## Results

There is no time during the year where the Azan isn&#39;t occurring somewhere on the globe.\*

Based on the experiment above, checking every minute of every day of the year, we find that there is **no minute** where the Azan isn&#39;t occurring somewhere on Earth. There are 11 (non-contiguous) minutes in the whole year where it occurs at only three cities, but I have manually verified the presence of a mosque **in** or **near** those cities.

There we have it! An answer to a question I had a decade ago. Keeping in mind we made some assumptions this ofcourse isn&#39;t a **definitive** answer, but it&#39;s a good enough approximation for me to be confident that the results are meaningful!

## Source Code

All source code and links to datasets are available in this github repository.
