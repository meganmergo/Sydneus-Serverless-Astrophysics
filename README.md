*Sydneus* makes tens to hundreds billion of procedurally generated suns, planets and moons in a virtually unlimited number of persistent galaxies.

This API's purpose is first and foremost to optimize the number and length of calls (hence the costs) you make to our serverless back-end, then to provide features like JSON-formatting, per-user throttling and per-user billing (if you have paying customers).

In a nutshell, we:
- Provide physical characteristics, orbital parameters and rotation (spin) parameters of celestial bodies
- Calculate the real-time state vector of each planet and each moon (suns are static in our galaxies)

A few useful notes:
- Galaxies are generated in 2D (meaning that a celestial body's state vector has no *inclination*, and that its rotation and revolution axis are always vertical to the galactic plane)
- All serverless results are cached in Redis 
- Some long running tasks on the serverless side can be shortened with **proof of work** (PoW, see below)

We have two programs:
- **sydneus3.py** is the front-end REST API to the back-end serverless generator hosted in *Azure function* and *Amazon lambda*.
- **app/locator.py** is an optional tool that leverages sydneus3.py to help perform operations on celestial bodies: please refer to the **app/** README for more information.

What we will NOT do for you:
- Authenticate your users
- Authorize access for your users (can user X see sun Y if he has not visited it yet ?)
- Provide a GUI
- Persist cached data (filesystem, database)
- Handle security (this API is not supposed to accessible from the Internet, but to be called from a web server or from an application server)

August 13, 2018: Added URL timeout
August 07, 2018: Fixed an issue with HTTP status codes
June 08, 2018: Fixed an issue with black holes habitable zone
June 25, 2018: Increased precision of orbital speed

## Getting started

### Recommended installation modes

We encourage you to consider one of the following installation modes:
- Ubuntu vanilla (for test)
- Ubuntu + HAproxy + nginx + UWSGI (for unmanaged production)
- Amazon Fargate (for managed production)
- Azure Kubernetes AKS (for managed production)

### Installation (Ubuntu vanilla)

### Python 3.5
sudo apt-get update

sudo apt-get install -y redis-server python3 curl python3-redis python3-pip

sudo pip3 install flask

clone **Sydneus** from GitHub

run **sydneus3.py** in daemon mode with the provided script: start3.sh

### Get your own access key
Azure and Amazon charge their customers, me included, so you would need to private message https://www.reddit.com/user/freevar to purchase an access key from **freevariable** to cover usage of our serverless backend in **pay per use** mode (so it's very, very cheap starting less than a dollar a month).

### Configuration
In your local clone directory, you need to create a file called localconf.py containing the following:
```python
ASKYOURS='your access key'
SEED='a random string of your liking that is unique for each galaxy'
```

### Run
By default, the server will start on localhost port 5043. You can set the port with option --port. Although you can run it standalone for development purposes, in production you are strongly advised to manage your front-end with UWSGI (ubuntu packages: uwsgi,uwsgi-core,uwsgi-emperor,uwsgi-plugin-python):

```
./sydneus3.py --port=5043 &
* Running on http://127.0.0.1:5043/ (Press CTRL+C to quit)
```

### Probe your first solar system!
Make the following call to the **list/sector** API; it will dump a list of systems in a JSON response:
```
curl 'http://127.0.0.1:5043/v1/list/sector/player4067/310/224'
```

## Design
### Locators
#### Astronomical bodies
Each galaxy is elliptical, with highest stars density near the core. The galaxy is divided into 1400x1400 sectors, each sector is a square covering 9 light years wide. So the galaxy is roughly 12600 light years wide.

- Galaxies are identified by their seed.
- Within a galaxy, sectors are identified by their cartesian coordinates separated with a column. For example, **345:628** corresponds to the sector located at x=345, y=628. Coordinates origin are the top left corner of the galaxy.
- Suns are identified by their trigram, which is unique within a given sector. For example, **345:628:Apo** corresponds to the Apo sun (if it exists in your galaxy, depending on the seed you have chosen!) within sector 345:628
- Planets are identified by their rank, the first one being closest to their sun. For example, **345:628:Apo:3** is the third planet in system Apo.
- Moons are identified by their rank, the first one being closest to their parent planet. For example, **345:638:Apo:3:6** is the sixth moon of planet 3 in the Apo system.

#### Idle spacecrafts 
Artificial bodies are not managed by Sydneus, but the API supports them as a convenience for ease of integration within your code logic.
Each spacecraft must be identified by a unique string of your choice, beginning with upper or lowercase ASCII. You must make sure this string is unique among all your users.

- Vessel *Harfang* orbiting sun Apo is located with **345:628:Apo:Harfang**
- Station *Cromwell* orbiting the fifth moon of planet 2 in the 4FN system is located with **76:578:4FN:2:5:Cromwell**

#### Accelerating spacecrafts
We provide no support for bodies under impulsion. You may consider them as a transition between two idle states (two different locators).

## API Documentation
We have four sets of APIs: procedural generation, realtime elements, cartography and management interface.
- Procedural generation is fully cacheable. It is cached in a redis DB called dataPlane.
- Realtime elements are not cached. They are interpolated from procedural generation items.
- Cartography elements are not cached. 
- Management data are cached in a redis DB called controlPlane.

APIs of type **/v1/list/** return a JSON array (ie, a list of JSON objects).
APIs of type **/v1/get/** return a JSON object (ie, an unordered list of key/value pairs).

### Procedural generation 

#### Parameters types

- sectorX: X locator of a sector (integer in range 1-1400)
- sectorY: Y locator of a sector (integer in range 1-1400)
- trigram: star's trigram (string)
- radius: a distance, in light years (float)
- pl: a planet's rank within a solar system (integer, 1=closest to the sun)
- mo: a moon's rank within a planetary system (integer, 1=closest to the planet)
- user: a unique user id that you manage (string)
- spacecraft: a unique ship or station id that you manage (string)

#### Epoch
The time at which physical parameters were set (especially the mean anomaly and the day progress) is 0.0 by convention.

#### Sectors

##### Request parameters

```
/v1/list/sector/<user>/<sectorX>/<sectorY>
```

##### Response elements

A JSON **list** of 0 or more suns with the following elements:

|Key       | Value   | Comment                                            |
|----------|---------|----------------------------------------------------|
|xly       |3.73524  | X location (in light years) within sector          |
|yly       |0.18146  | Y location (in light years) within sector          |
|trig      |rka      | Trigram within sector                              |
|seed      |83455592 | Seed                                               |
|cls       |6        | Spectral class                                     |

#### Discs

##### Request parameters

```
/v1/list/disc/<user>/<sectorX>/<sectorY>/<trigram>/<radius>
```

##### Response elements

A JSON **list** of 0 or more suns with the following elements:

|Key       | Value   | Comment                                            |
|----------|---------|----------------------------------------------------|
|dist      |2.85539  | Distance to sun reference                          |
|xly       |3.73524  | X location (in light years) within sector          |
|yly       |0.18146  | Y location (in light years) within sector          |
|sectorX   |145      | Sector X of star (not always the same as sun ref)  |
|sectorY   |608      | Sector Y of star (not always the same as sun ref)  |
|trig      |rka      | Trigram within sector                              |
|seed      |83455592 | Seed                                               |
|cls       |6        | Spectral class                                     |


#### Stars

##### Request parameters

```
/v1/get/su/<user>/<sectorX>/<sectorY>/<trigram>
```

##### Response elements

The physical characteristics of the sun:

|Key       | Value   | Comment                                            |
|----------|---------|----------------------------------------------------|
|trig      |Apo      | Star name, third component of the star locator     |
|x         |300      | Sector X, first component of the star locator      |
|y         |650      | Sector Y, second component of the star locator     |
|xly       |3.6083   | X location within sector (in light years)          |
|yly       |8.03151  | Y location within sector (in light years)          |
|lumiSU    |0.88102  | Solar luminosity (Our sun = 1.0)                   |
|absMag    |-1.6492  | Absolute magnitude                                 |
|mSU       |1.02635  | Solar mass (our sun = 1.0)                         |
|nbPl      |6        | Number of orbiting planets                         |
|HZcenterAU|2.30302  | Radius of the center of the Habitable Zone (in AU) |
|cls       |3        | Spectral class (Sydneus code, see below)           |
|spectral  |K        | Spectral class (see below)                         |
|spin      |1497.87  | Rotation time (in seconds)                         |
|seed      |67403928 | Star seed                                          |
|irrOuterAU|25.67362 | Outer radius (in AU) of the radiation zone         |

##### Spectral classes

|cls       | spectral   | Comment                                            |
|----------|------------|----------------------------------------------------|
|1         |M           |                                                    |
|2         |K           |                                                    |
|3         |G           |                                                    |
|4         |F           |                                                    |
|5         |A           |                                                    |
|6         |B           |                                                    |
|7         |O           |                                                    |
|8         |bd          |brown dwarf                                         |
|9         |wd          |white dwarf                                         |
|10        |ns/bh       |Neutron star or black hole                          |


#### Planets

##### Request parameters

```
/v1/list/pl/<user>/<sectorX>/<sectorY>/<trigram>
```

##### Response elements

A JSON **list** of 0 or more planets with the following elements:

|Key       | Value   | Comment                                            |
|----------|---------|----------------------------------------------------|
|rank      |11       | Planet rank within system                          |
|nbMo      |5        | Quantity of moons                                  |
|cls       |J        | Terran     (E) or Jovian (J)                       |
|g         |12.6057  | Surface gravity  (Earth=9.81)                      |
|mEA       |5.36242  | Earth mass (Earth=1.0)                             |
|radEA     |2.0416   | Earth radius (Earth=1.0)                           |
|denEA     |0.62789  | Earth density (Earth=1.0)                          |
|hasAtm    |True     | Has an atmosphere                                  |
|isLocked  |True     | Spin is locked                                     |
|isIrr     |False    | Is within sun radiation zone                       |
|inHZ      |False    | Is in Habitable Zone                               |
|smiAU     |763.662  | Semi-minor axis in AU                              |
|smaAU     |763.675  | Semi-major axis in AU                              |
|ecc       |0.00574  | Eccentricity                                       |
|per       |0.87874  | Periapsis (in radians)                             |
|ano       |4.42784  | Anomaly at epoch                                   |
|hill      |153...   | Radius of Hill sphere (in km)                      |
|roche     |153...   | Roche limit in km)                                 |
|spin      |-0.1901  | Rotation (Earth=1.0=24h). Negative for retrograde  |
|period    |453...   | Orbital period in seconds                          |
|dayProg   |0.89799  | Day progress at epoch (1.0=midnight) ref meridian  |

#### Moons

##### Request parameters

```
/v1/list/mo/<user>/<sectorX>/<sectorY>/<trigram>/<pl>
```

##### Response elements

A JSON **list** of 0 or more moons with the following elements:

|Key       | Value   | Comment                                            |
|----------|---------|----------------------------------------------------|
|rank      |11       | Moon rank within planetary system                  |
|cls       |J        | Terran     (E) or Jovian (J)                       |
|g         |12.6057  | Surface gravity  (Earth=9.81)                      |
|mEA       |5.36242  | Earth mass (Earth=1.0)                             |
|radEA     |2.0416   | Earth radius (Earth=1.0)                           |
|denEA     |0.62789  | Earth density (Earth=1.0)                          |
|hasAtm    |True     | Has an atmosphere                                  |
|isLocked  |True     | Spin is locked                                     |
|isIrr     |False    | Is within sun radiation zone                       |
|inHZ      |False    | Is in Habitable Zone                               |
|smiAU     |763.662  | Semi-minor axis in AU                              |
|smaAU     |763.675  | Semi-major axis in AU                              |
|ecc       |0.00574  | Eccentricity                                       |
|per       |0.87874  | Periapsis (in radians)                             |
|ano       |4.42784  | Anomaly at epoch                                   |
|hill      |153...   | Radius of Hill sphere (in km)                      |
|roche     |153...   | Roche limit in km)                                 |
|spin      |-0.1901  | Rotation (Earth=1.0=24h). Negative for retrograde  |
|period    |453...   | Orbital period in seconds                          |
|dayProg   |0.89799  | Day progress at epoch (1.0=midnight) ref meridian  |


### Realtime API

#### Parameter types

They are the same as above.

#### Stars
In Sydneus, **stars are static**.

#### Planets

##### Request parameters

```
/v1/get/pl/elements/<user>/<sectorX>/<sectorY>/<trigram>/<pl>
```

##### Response elements

|Key            | Value     | Comment                                            |
|---------------|-----------|----------------------------------------------------|
|spinFormatted  |-4h33m44s  |Negative means retrograde                           |
|dayProgress    |0.89799    |Day progress now (same as at epoch if locked)       |
|localTime      |4h5m49s    |Time now at ref meridian                            |
|fromPer        |2238y      |Time from periapsis                                 |
|to  Per        |12130y     |Time to periapsis                                   |
|progress       |15.58%     |Orbital period progress                             |
|meanAno        |4.436726   |Mean Anomaly (realtime) in radians                  |
|rho            |11442...   |Polar distance from sun (realtime) in km            |
|theta          |1.857695   |Polar angle (realtime) in radians                   |
|v              |2.460833   |Orbital speed in km/s                               |


#### Moons

##### Request parameters

```
/v1/get/mo/elements/<user>/<sectorX>/<sectorY>/<trigram>/<pl>/<mo>
```

##### Response elements

|Key            | Value     | Comment                                            |
|---------------|-----------|----------------------------------------------------|
|spinFormatted  |-4h33m44s  |Negative means retrograde                           |
|dayProgress    |0.89799    |Day progress now (same as at epoch if locked)       |
|localTime      |4h5m49s    |Time now at ref meridian                            |
|fromPer        |2238y      |Time from periapsis                                 |
|to  Per        |12130y     |Time to periapsis                                   |
|progress       |15.58%     |Orbital period progress                             |
|meanAno        |4.436726   |Mean Anomaly (realtime) in radians                  |
|rho            |11442...   |Polar distance from sun (realtime) in km            |
|theta          |1.857695   |Polar angle (realtime) in radians                   |
|v              |2.460833   |Orbital speed in km/s                               |

## Examples

### Procedural generation

#### Sectors
Exemple: generate all stars in sector 310:224 on behalf of *player4067*:

```
curl 'http://127.0.0.1:5043/v1/list/sector/player4067/310/224'
```

#### Discs
Example: generate a disc of radius 3.4 light years centered around star 400:29:jmj on behalf of user *player4067*

```
curl 'http://127.0.0.1:5043/v1/list/disc/player4067/400/29/jmj/3.4'
```

#### Suns (without Proof of Work)
Example: generate the physical characteristics of sun RWh in sector 400:29 (on behalf of player4067). Here, we see that this sun has only one planet in orbit.
Also notice the proof of work that we may reuse later on.

```
curl 'http://127.0.0.1:5043/v1/get/su/player4067/400/29/RWh'

{"pow": "JRprDMexJidlAbtrgsN7tpIlqOxy4b8lRa7h5hiRqZE=", "trig": "RWh", "perStr": 5.705832, "per": 3.5505651852343463, "lumiSU": 1.1010247142796168, "nbPl": 1, "HZcenterAU": 1.303023247848569, "seed": 91106006, "cls": 3, "spectral": "G", "xly": 1.423, "y": 29, "x": 400, "yly": 8.031, "mSU": 1.026352406, "spin": 1254697.8796800002}
```

#### Suns (with Proof of Work)
Example: same as above, however here you will need not only the sun locator (400:29:RWh), but also the proof of work, the spectral class code (3), the seed, and the x and y coordinates (in light years) of RWh in sector 400:29. All this can be re-used from the previous call above.

```
curl 'http://127.0.0.1:5043/v1/get/su/player4067/400/29/RWh/91106006/3/1.423/8.031/JRprDMexJidlAbtrgsN7tpIlqOxy4b8lRa7h5hiRqZE='

{"perStr": 5.705832, "trig": "RWh", "lumiSU": 1.1010247142796168, "per": 3.5505651852343463, "yly": 8.031, "nbPl": 1, "HZcenterAU": 1.303023247848569, "seed": "91106006", "xly": 1.423, "y": 29, "x": 400, "spin": 1254697.8796800002, "mSU": 1.026352406, "cls": 3, "spectral": "G"}
```

#### Planets (without Proof of Work)

Example: generate the physical characteristics of all planets orbiting sun RWh. We can confirm that there is only one planet because the returned list has only one item. It is a small planet (radius 1487.5km) in the habitable zone of RWh, what's more it has an atmosphere but its gravity is low, similar to our good old Moon.

```
curl 'http://127.0.0.1:5043/v1/list/pl/player4067/400/29/RWh'

[{"mEA": 0.010402687467665962, "hasAtm": true, "smiAU": 1.4784174519337556, "ano": 0.9587135469107532, "period": 55880562.18270369, "spin": 0.09075012880266921, "dayProgressAtEpoch": 0.2056876, "perStr": 4.812696000000001, "per": 3.902947712776953, "isLocked": false, "hill": 466355.68892496126, "inHZ": true, "cls": "E", "ecc": 0.024690300000000002, "denEA": 0.817121, "radEA": 0.23322007389358487, "g": 1.873998154697319, "smaAU": 1.478868287777208, "isIrr": false, "rank": 1}]
```

#### Planets (with Proof of Work)

Example: Same as above, but this time reusing the PoW and the same procedurally generated items obtained above during sun generation.
```
curl 'http://127.0.0.1:5043/v1/list/pl/player4067/400/29/RWh/1/91106006/3/1.423/8.031/JRprDMexJidlAbtrgsN7tpIlqOxy4b8lRa7h5hiRqZE='

[{"mEA": 0.010402687467665962, "hasAtm": true, "smiAU": 1.4784174519337556, "ano": 0.9587135469107532, "period": 55880562.18270369, "spin": 0.09075012880266921, "dayProgressAtEpoch": 0.2056876, "perStr": 4.812696000000001, "per": 3.902947712776953, "isLocked": false, "hill": 466355.68892496126, "inHZ": true, "cls": "E", "ecc": 0.024690300000000002, "denEA": 0.817121, "radEA": 0.23322007389358487, "g": 1.873998154697319, "smaAU": 1.478868287777208, "isIrr": false, "rank": 1}]
```

#### Moons (without Proof of Work)

Example: generate the physical characteristics of all moons of the third planet orbiting sun 9w3. There are three jovian (J) moons ranked from 1 to 3

curl 'http://127.0.0.1:5043/v1/list/mo/player4067/198/145/9w3/3'

[{"mEA": 1.3922042044673781, "hasAtm": true, "smiAU": 0.00937585546978511, "ano": 3.2488277772276413, "period": 3047605.525047631, "rank": 1, "dayProgressAtEpoch": 0.23665009, "per": 2.4477264163362618, "roche": 56452.037479146515, "isLocked": false, "hill": 327403.42420291057, "smi": 1402608.0170835573, "inHZ": false, "sma": 1405978.2123213192, "cls": "J", "ecc": 0.069197885, "denEA": 0.0851810388, "radEA": 2.5347368107726855, "spin": 0.018242621667133474, "g": 2.123208588612319, "smaAU": 0.009398383833425806, "isIrr": false}, {"mEA": 0.1008820599462187, "hasAtm": true, "smiAU": 0.029347286498631237, "ano": 3.2488277772276413, "period": 16876963.99760388, "rank": 2, "dayProgressAtEpoch": 0.28218672, "per": 2.4477264163362618, "roche": 33001.31945826333, "isLocked": false, "hill": 427245.27185665147, "smi": 4390291.579822278, "inHZ": false, "sma": 4400840.599644272, "cls": "J", "ecc": 0.069197885, "denEA": 0.4263719076, "radEA": 0.6177621873100944, "spin": 0.05851302343592033, "g": 2.590161017377747, "smaAU": 0.029417802340544485, "isIrr": false}, {"mEA": 3.1024952303922166, "hasAtm": true, "smiAU": 0.09185969510817159, "ano": 3.2488277772276413, "period": 93460886.40260153, "rank": 3, "dayProgressAtEpoch": 0.89569314, "per": 2.4477264163362618, "roche": 51514.68357163969, "isLocked": false, "hill": 4189865.0970187383, "smi": 13742014.818891585, "inHZ": false, "sma": 13775034.217280721, "cls": "J", "ecc": 0.069197885, "denEA": 0.1120956618, "radEA": 3.0212521653480877, "spin": 0.253858158043709, "g": 3.3303711697373335, "smaAU": 0.09208041615298604, "isIrr": false}]

#### Moons (with Proof of Work)

Same as above, but you would append the PoW obtained during sun 9w3 generation.

#### Spacecrafts

We do not provide a **list/spacecraft** API, because we intend our API to be highly scalable. If you have tens of thousands of users, listing all spacecrafts orbiting a given sun would be a very bad architecture design.

##### Spacecraft in solar orbit

```
curl 'http://127.0.0.1:5043/v1/get/spacecraft/player4067/400/29/RWh/Cromwell'
```

##### Spacecraft in planet orbit

```
curl 'http://127.0.0.1:5043/v1/get/spacecraft/player4067/400/29/RWh/3/Cromwell'
```

##### Spacecraft in moon orbit

```
curl 'http://127.0.0.1:5043/v1/get/spacecraft/player4067/400/29/RWh/3/2/Cromwell'
```

### Realtime elements API

#### Real time orbital elements of a planet
Example: get orbital elements of the planet orbiting sun RWh. The planet's rank 1 must be provided, even if it's alone in its solar system.
```
curl 'http://127.0.0.1:5043/v1/get/pl/elements/player4067/400/29/RWh/1'

{"spinFormatted": "2h10m40s", "fromPer": "149d5h", "dayProgress": 0.3178352585976779, "toPer": "1y132d", "meanAno": 3.8622080878729736, "rho": 225397850.12794173, "progress": "23.08%", "localTimeFormatted": "41m32s", "theta": 2.4530273644516596, "periodFormatted": "1y281d", "localTime": 2492.086232658437}
```

#### Spacecrafts

Unlike celestial bodies, the epoch is not 0.0 but the time when the spacecraft started its orbit.

(To be completed...)

### Cartography API

#### Solar system map and image
Get the planetary distribution and SVG rendering of system 9w3 over a logarithmic scale spread between pixels 10 and 300

```
curl 'http://127.0.0.1:5043/v1/map/su/player4067/10/300/198/145/9w3'

{"logScale": [{"span": 10.0, "rank": 1}, {"span": 134.57648568274683, "rank": 2}, {"span": 196.49873621842647, "rank": 3}, {"span": 204.26638450469608, "rank": 4}, {"span": 225.0497219879097, "rank": 5}, {"span": 233.73016686825028, "rank": 6}, {"span": 248.63731668832162, "rank": 7}, {"span": 287.8319534128117, "rank": 8}, {"span": 300.0, "rank": 9}], "svg": {}}
```

### Management API
#### List users
Example: list all users which have been billed so far.
```
curl 'http://127.0.0.1:5043/v1/list/users'
```
#### Show detailed service consumption 
Example: show all billing dots for user player4067
```
curl 'http://127.0.0.1:5043/v1/list/billing/player4067'
["{'verb': 'plGen', 't': 1524054485.730217, 'result': 200}", "{'verb': 'plMap', 't': 1524054216.033575, 'result': 200}"]

```
