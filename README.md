![](https://i.imgur.com/FtbtTn5.png)
===

[1 General Information](https://github.com/gerberox/dcity/blob/master/README.md#1-general-information)

## 1 General Information

All game data is stored on hive-engine, so you will need to query engine to build up your interface/application. Basic py example:

```py
def engine_everything(contract, table, query, offset):
    url = 'https://api.hive-engine.com/rpc/contracts'
    params = {'contract':contract, 'table':table, 'query':query, 'limit':1000, 'offset':offset, 'indexes':[]}
    j = {'jsonrpc':'2.0', 'id':1, 'method':'find', 'params':params}

    with requests.post(url, json=j) as r:
        data = r.json()
        result = data['result']
        if len(result) == 1000:
            result += engine_everything("nft", "CITYinstances", query, offset+1000)
    return result
    
query1 = {"account": "gerber"}
my_city = engine_everything("nft", "CITYinstances", query1, 0)
```

as you can see there are 3 main variables you going to use to get data:
* contract
* table
* query

More examples:
```py
tradehistory24h = engine_everything("nftmarket", "CITYtradesHistory", {}, 0)
NFTs_by_property = engine_everything("nft", "CITYinstances", {"properties.name":"Basic Home"}, 0)
```

2 Complete Math for one city
===

2.1 Starting Point - data
---

To display City for a player with proper numbers you need to collect data from 3 sources:

actual CITY NFT collection for player:
```py
my_city = engine_everything("nft", "CITYinstances", {"account":username}, 0)
```

Government rules for game:
```py
gov_api = engine_everything("nft", "APIinstances", {"_id":1}, 0)
```
Player Card in Api for events:
```py
player_api = engine_everything("nft", "APIinstances", {"properties.x":username}, 0)
```

2.2 Building up Base Numbers
---
You already have all the NFT's for player, you can sum it by property for base numbers for:
* population
* popularity
* income
* creativity
* education

Creativity and education are strings cause mistake in creation process :( that can't be reversed.
These numbers are not a final numbers for city just base for calculation.

```py
population = sum([b['properties']['population'] for b in my_city if 'population' in b['properties']])
popularity = sum([b['properties']['popularity'] for b in my_city if 'popularity' in b['properties']])
income = sum([b['properties']['income'] for b in my_city if 'income' in b['properties']])
creativity = sum([float(b['properties']['creativity']) for b in my_city if 'creativity' in b['properties']])
education = sum([float(b['properties']['education']) for b in my_city if 'education' in b['properties']])
```

Good thing is also to build summary on assets, as it can be useful for applying multipliers:

```py
summary = {}
for asset in my_city:
    if asset['properties']['name'] in summary:
        summary[asset['properties']['name']] += 1
    else:
        summary[asset['properties']['name']] = 1
```
2.3 Game Multipliers
---
All the multipliers:
```py
education_multiplier = 1
creativity_multiplier = 1
promote_eco = False
income_multiplier = 1
population_multiplier = 1
```

2.4 Building up Government Multipliers
---
Government rules for game:
```py
gov_api = engine_everything("nft", "APIinstances", {"_id":1}, 0)
```

Inside gov_api, property info cantain data to build up multipliers, you might need to parse it cause it is string. Inside info dict are 2 keys, tax and pf. pf means police fund and we can ignore it when doing math for city.

tax contains list of taxes, and at the same time bonuses.

```json
{
    "income_tax" : info_string['tax'][0],    # tax based on SIM price
    "police_tax" : info_string['tax'][1],    # tax to fund police
    "edu_tax" : info_string['tax'][2],       # tax to fund education
    "art_tax" : info_string['tax'][3],       # tax to fund creativity
    "war_tax" : info_string['tax'][4],       # tax to fund war
    "eco_tax" : info_string['tax'][5],       # tax for eco program
    "job_tax" : info_string['tax'][6],       # tax for jobs program
    "basic_tax" : info_string['tax'][7],     # tax for nothing, basic tax
    "rich_tax" : info_string['tax'][9],      # tax on top 3 or 5 players
    }
```

``"income_tax" : info_string['tax'][0],    # tax based on SIM price``
This one os based on average SIM token price in last 3 days, when price goes down, this tax should go up and decrease inflation.

President chosen by players can decide about some game bonuses, but those are not for free.
President can boost education, creativity and income from Eco assets.
So from that info to do proper math on game we need 4 things:
* sum on tax:
```py
tax = sum(info_string['tax']) - info_string['tax'][9]
```

```py
## education multiplier

if info_string['tax'][2] == 5:
    education_multiplier + 0.1

## creativity multiplier

if info_string['tax'][3] == 5:
    creativity_multiplier + 0.1

## Eco Program

if info_string['tax'][5] == 5:
    promote_eco = True
```

Comment: tax on richest is there but may be never used

2.5 Building up Technology Multipliers
---
Now we need to modify our multipliers by numbers from technologies.
Thanks to summary we applying bonuses only once.

+5% bonus to education from ``Free Internet Connection`` and +10% from ``Open Source``:
```py
if "Free Internet Connection" in summary: #2-1
    education_multiplier += 0.05
if "Open Source" in summary: #2-3
    education_multiplier += 0.10
```
Changes in popularity due to technologies:

```py
if "GMO Farming" in summary: #4-1
    if 'Farm' in summary:
        popularity -= (summary['Farm']*3)

if "Cold Fusion" in summary: #4-3
    if 'Nuclear Plant' in summary:   
        popularity += (summary['Nuclear Plant']*15)
```

Income multiplier from technology ( only highest bonus is working ):
```py
if "Mining Operation - HIVE" in summary: #3-4
    income_multiplier = 1.02
if "Mining Operation - ETH" in summary: #3-4
    income_multiplier = 1.04
if "Mining Operation - BTC" in summary: #3-4
    income_multiplier = 1.06
```
Check if player have full branch of tech Control, Education, Automation or Utility for population multiplier:
```py
all_tech = [["Police Equipment" , "Drone Technology", "RoboCop", "Neural Network", "Voice to Skull"],
         ["Free Internet Connection", "Better Documentation Practice", "Open Source", "Free Education", "Advanced Training"],
         ["Basic Automation", "Fully Automated Brewery", "AI Technology", "Mining Operation - BTC", "Advanced Robotics"],
         ["GMO Farming", "ECO Energy", "Cold Fusion", "Dyson Sphere", "Advanced Recycling"]]

for listx in all_tech:
    tech_check =  all(item in listx for item in summary)
    if tech_check is True:
        population_multiplier += 0.01
```

2.6 Additional Cards effects
---
Garbage Dump have negative popularity which is reversed if player have more than 10 different cards ( will be increased to 30 after update release):
```py
if 'Garbage Dump' in summary and len(set(summary)) > 25:
    popularity += summary['Garbage Dump'] * 60
```

Hospital give 1% bonus to population but only for one in assets:
```py
if 'Hospital' in summary:
    population_multiplier += 0.01
```
Tax reduction thanks to Law Firm:
```py
if 'Law Firm' in display and tax > 0:
    tax = 0.9*tax
```


2.7 Applying bonuses from events
---
Now from our players API we have:
```py
player_api = engine_everything("nft", "APIinstances", {"properties.x":username}, 0)
```

with two properties e and e2 to parse for events
```py
e = {1: [1, 1], 2: [1, 1], 3: [1, 1]}
e2 = {4: [1, 1], 5: [1, 1], 6: [1, 1]}
```
first number in list for event = bonus
second number in list for event = start time in epoch time

we have 6 possible events players can organize ( actually 4 in game )
1 - BEER Fest - bonus to popularity
2 - Science Convention - bonus to education
3 - Art Convention - bonus to creativity
4 - WEED Fest - bonus to creativity


We can check them 1 by 1:
```py
if e[1][1] > time.time()-(24*60*60):    ## event last for 24h
    popularity += e[1][0]               ## applying bonus for BEER fest

if e[2][1] > time.time()-(24*60*60):    ## event last for 24h
    education += e[1][0]               ## applying bonus for Science Convention

if e[3][1] > time.time()-(24*60*60):    ## event last for 24h
    creativity += e[1][0]               ## applying bonus for Art convention

if e2[4][1] > time.time()-(24*60*60):    ## event last for 24h
    creativity += e[1][0]               ## applying bonus for WEED Fest
```


2.8 Final Population
---
We have our base population from cards now to calculate final population we need to apply popularity effect:
``popularity_effect = 1+((popularity**0.70)/100)``

```py
population = round(population*(1+((max(0, popularity)**0.70)/100)))
```

and apply population_multiplier we calculated before ( max possible is 5%, 1% from hospital and 4 x 1% from collecting full branch of tech)

```py
final_population = round(population_multiplier*(population*(1+((max(0, popularity)**0.70)/100))))
```

2.9 Final Income and Unemplyoment
---
This is probably hardest part in calculations , as we need to check every single asset if City have enough workers for it then also apply bonuses from technologies and government.
As starting point we have income from cards, now we going to decrease it if there is not enough workers else add bonuses.
we need to create social_aid variable to finalize social income

```py
social_aid = 0
workers = 0
    ## CHECKING ALL THE ITEMS FOR ACCOUNT FOR FINAL INCOME AND UNEMPLOYMENT
for asset in my_city:
    if 'workers' in asset['properties']:
        add_workers = asset['properties']['workers']
        ## TECHNOLOGY ON FACTORY
        if "Advanced Robotics" in summary and asset['properties']['name'] == 'Factory': #3-5
            add_workers = 2
        if "AI Technology" in summary: #3-3
            if asset['properties']['name'] == 'Factory':
                add_workers = add_workers//2

        workers += add_workers
        ## REDUCE INCOME IF NOT ENOUGH WORKERS
        if workers > final_population:
            if 'income' in asset['properties']:
                income -= asset['properties']['income']
            if 'education' in asset['properties']:
                education -= float(asset['properties']['education'])
        ## OR ADD BONUSES FROM TECH AND OTHER
        else:
            if asset['properties']['name'] == 'Solar Panel' and "Dyson Sphere" in summary:
                income += 8
            if asset['properties']['name'] == 'Garbage Dump' and "Advanced Recycling" in summary:
                income += 15
            if asset['properties']['name'] == 'Farm' and "GMO Farming" in summary:
                income += 4
            if asset['properties']['name'] == 'Wind Turbine' and "ECO Energy" in summary:
                income += 2
            if asset['properties']['name'] == 'Wind Turbine' and promote_eco == True:
                income += 2
            if asset['properties']['name'] == 'Solar Panel' and "ECO Energy" in summary:
                income += 2
            if asset['properties']['name'] == 'Solar Panel' and promote_eco == True:
                income += 2
            if asset['properties']['name'] == 'Factory' and "Basic Automation" in summary:
                income += 3
            if asset['properties']['name'] == 'University' and "Free Education" in summary:
                education += 10
            if asset['properties']['name'] == 'Art Gallery':
                income += creativity/100
            if asset['properties']['name'] == 'Social Aid Office':
                social_aid += 100
```
Now we need to reduce income by social support costs( -0.2 for unemployment and costs reduced by Social Aid Offices = social_aid):

```py
income -= max(0, round(((final_population-workers)*0.2)) - social_aid)
```
At the end multiplier to finalize and taxes we took from goc_api before at the end:
```py
final_income = round((1 - tax/100) * income_multiplier * income)
```
if you want to display unemployment, social support costs:
```py
unemployment = final_population-workers
social_support = max(0, round(((final_population-workers)*0.2)) - social_aid))
```

## 2.10 Final Education and Creativity
We already have multipliers for education adn creativity from technologies and gov, to get final numbers we just need to apply it:
```py
final_education = education*education_multiplier
final_creativity = creativity*creativity_multiplier
```

## 2.11 All in one

```py
import requests
import json
import time
import ast

def engine_everything(contract, table, query, offset):
    url = 'https://api.hive-engine.com/rpc/contracts'
    params = {'contract':contract, 'table':table, 'query':query, 'limit':1000, 'offset':offset, 'indexes':[]}
    j = {'jsonrpc':'2.0', 'id':1, 'method':'find', 'params':params}

    with requests.post(url, json=j) as r:
        data = r.json()
        result = data['result']
        if len(result) == 1000:
            result += engine_everything("nft", "CITYinstances", query, offset+1000)
    return result

username = "gerber"

## DATA
query1 = {"account": username}
my_city = engine_everything("nft", "CITYinstances", query1, 0)
gov_api = engine_everything("nft", "APIinstances", {"_id":1}, 0)
player_api = engine_everything("nft", "APIinstances", {"properties.x":username}, 0)

## BASE NUMBERS

population = sum([b['properties']['population'] for b in my_city if 'population' in b['properties']])
popularity = sum([b['properties']['popularity'] for b in my_city if 'popularity' in b['properties']])
income = sum([b['properties']['income'] for b in my_city if 'income' in b['properties']])
creativity = sum([float(b['properties']['creativity']) for b in my_city if 'creativity' in b['properties']])
education = sum([float(b['properties']['education']) for b in my_city if 'education' in b['properties']])

## SUMMARY
summary = {}
for asset in my_city:
    if asset['properties']['name'] in summary:
        summary[asset['properties']['name']] += 1
    else:
        summary[asset['properties']['name']] = 1

## ALL MULTIPLIERS

education_multiplier = 1
creativity_multiplier = 1
promote_eco = False
income_multiplier = 1
population_multiplier = 1

## GOV MULTIPLIERS
info_string = ast.literal_eval(gov_api[0]['properties']['info'])
## TAX
tax = sum(info_string['tax']) - info_string['tax'][9]

## education multiplier
if info_string['tax'][2] == 5:
    education_multiplier + 0.1

## creativity multiplier
if info_string['tax'][3] == 5:
    creativity_multiplier + 0.1

## Eco Program
if info_string['tax'][5] == 5:
    promote_eco = True

## TECH MULTIPLIERS
if "Free Internet Connection" in summary: #2-1
    education_multiplier += 0.05

if "Open Source" in summary: #2-3
    education_multiplier += 0.10

if "GMO Farming" in summary: #4-1
    if 'Farm' in summary:
        popularity -= (summary['Farm']*3)

if "Cold Fusion" in summary: #4-3
    if 'Nuclear Plant' in summary:   
        popularity += (summary['Nuclear Plant']*15)

if "Mining Operation - HIVE" in summary: #3-4
    income_multiplier = 1.02
if "Mining Operation - ETH" in summary: #3-4
    income_multiplier = 1.04
if "Mining Operation - BTC" in summary: #3-4
    income_multiplier = 1.06

all_tech = [["Police Equipment" , "Drone Technology", "RoboCop", "Neural Network", "Voice to Skull"],
         ["Free Internet Connection", "Better Documentation Practice", "Open Source", "Free Education", "Advanced Training"],
         ["Basic Automation", "Fully Automated Brewery", "AI Technology", "Mining Operation - BTC", "Advanced Robotics"],
         ["GMO Farming", "ECO Energy", "Cold Fusion", "Dyson Sphere", "Advanced Recycling"]]

for listx in all_tech:
    tech_check =  all(item in listx for item in summary)
    if tech_check is True:
        population_multiplier += 0.01

## CARDS EFFECT
if 'Garbage Dump' in summary and len(set(summary)) > 10:
    popularity += summary['Garbage Dump'] * 60
if 'Hospital' in summary:
    population_multiplier += 0.01
if 'Law Firm' in display and tax > 0:
    tax = 0.9*tax

## EVENTS

e = ast.literal_eval(player_api[0]['properties']['e'])
e2 = ast.literal_eval(player_api[0]['properties']['e2'])

if e[1][1] > time.time()-(24*60*60):    ## event last for 24h
    popularity += e[1][0]               ## applying bonus for BEER fest

if e[2][1] > time.time()-(24*60*60):    ## event last for 24h
    education += e[1][0]               ## applying bonus for Science Convention

if e[3][1] > time.time()-(24*60*60):    ## event last for 24h
    creativity += e[1][0]               ## applying bonus for Art convention

if e2[4][1] > time.time()-(24*60*60):    ## event last for 24h
    creativity += e[1][0]               ## applying bonus for WEED Fest

## FINAL POPULATION
final_population = round(population_multiplier*(population*(1+((max(0, popularity)**0.70)/100))))

## FNAL INCOME
social_aid = 0
workers = 0
    ## CHECKING ALL THE ITEMS FOR ACCOUNT FOR FINAL INCOME AND UNEMPLOYMENT
for asset in my_city:
    if 'workers' in asset['properties']:
        add_workers = asset['properties']['workers']
        ## TECHNOLOGY ON FACTORY
        if "Advanced Robotics" in summary and asset['properties']['name'] == 'Factory': #3-5
            add_workers = 2
        if "AI Technology" in summary: #3-3
            if asset['properties']['name'] == 'Factory':
                add_workers = add_workers//2

        workers += add_workers
        ## REDUCE INCOME IF NOT ENOUGH WORKERS
        if workers > final_population:
            if 'income' in asset['properties']:
                income -= asset['properties']['income']
                
            if 'education' in asset['properties']:
                education -= float(asset['properties']['education'])
                
        ## OR ADD BONUSES FROM TECH AND OTHER
        else:
            if asset['properties']['name'] == 'Solar Panel' and "Dyson Sphere" in summary:
                income += 8
            if asset['properties']['name'] == 'Garbage Dump' and "Advanced Recycling" in summary:
                income += 15
            if asset['properties']['name'] == 'Farm' and "GMO Farming" in summary:
                income += 4
            if asset['properties']['name'] == 'Wind Turbine' and "ECO Energy" in summary:
                income += 2
            if asset['properties']['name'] == 'Wind Turbine' and promote_eco == True:
                income += 2
            if asset['properties']['name'] == 'Solar Panel' and "ECO Energy" in summary:
                income += 2
            if asset['properties']['name'] == 'Solar Panel' and promote_eco == True:
                income += 2
            if asset['properties']['name'] == 'Factory' and "Basic Automation" in summary:
                income += 3
            if asset['properties']['name'] == 'University' and "Free Education" in summary:
                education += 10
            if asset['properties']['name'] == 'Art Gallery':
                income += creativity/100
            if asset['properties']['name'] == 'Social Aid Office':
                social_aid += 100

income -= max(0, round(((final_population-workers)*0.2)) - social_aid)
final_income = round((1 - tax/100) * income_multiplier * income)

final_education = education*education_multiplier
final_creativity = creativity*creativity_multiplier

print(username, "population:", final_population, "popularity:", popularity, "income:", final_income, "education:", final_education, "creativity:", final_creativity)
```


3 Additional Informations
===

## 3.1 Chance for Training
Training is new every 24h feature on citizens:
First we need to grab our **gov_api** and check in taxes if Jobs Program is activated by President of the game:
```py
    if info_string['tax'][6] == 5:
        jobs_program = 2
    else:
        jobs_program = 1
``` 
Then we check for player's Job Centers in summary and calculate chance for training:
```py
    if 'Job Center' in summary:
        job_centers = summary['Job Center']
    else:
        job_centers = 0

    training_chance = min(100,((job_centers*3*jobs_program) + ((number_of_homeless + number_of_immigrants) * 0.01)))
```

If player have Advanced Training tech he have 2 draws every 24h


## 3.2 Technology Tree

In **player_api** you have property **tech** (remember data in API is always as string):

```py
tech = {'t': 111, 1: [0, 0, 0, 0, 0], 2: [0, 0, 0, 0, 0], 3: [0, 0, 0, 0, 0], 4: [0, 0, 0, 0, 0]}
```

t - is time of last initial research and base for cooldown
After unlocking technology player have 36 hours or 24 hours(with Better Documentation Practice tech) cooldown before he can unlock another one.

property tech contains  full Tech-Tree for a player:
```json=
tech_string = {1: [0, 0, 0, 0, 0], 2: [0, 0, 0, 0, 0], 3: [0, 0, 0, 0, 0], 4: [0, 0, 0, 0, 0]}
```

4 branches: Control, Education, Automation, Utility
Player need to unlock it from key[0] to key[5], can't jump to the last one in branch before unlocking previous levels.

```py
all_tech = [["Police Equipment" , "Drone Technology", "RoboCop", "Neural Network", "Voice to Skull"],
         ["Free Internet Connection", "Better Documentation Practice", "Open Source", "Free Education", "Advanced Training"],
         ["Basic Automation", "Fully Automated Brewery", "AI Technology", "Mining Operation - BTC", "Advanced Robotics"],
         ["GMO Farming", "ECO Energy", "Cold Fusion", "Dyson Sphere", "Advanced Recycling"]]
```


## 3.3 Chance for Technology

**Reminder**
To discover technology player need to have Research Center and unlock technology in Tech-Tree

Draw every 24h:
```py
if 'Research Center' in summary:
    tech_chance = round(min(25,education/40))
else:
    tech_chance = 0
```

## 3.4 Technologies

**Reminder**
To discover technology player need to have Research Center and unlock technology in Tech-Tree

Technologies are minable NFT, players can trade it.
For list of technologies for account:
```py
my_tech = engine_everything("nft", "CITYinstances", {"account":username, "properties.type":"tech"}, 0)
```

## 3.5 Government, Police Fund

Most important element of government is tax, as it decrease final income of players.
You can display it as sum and also show all the taxes > 0.

First lets grab gov_api:
```py
gov_api = engine_everything("nft", "APIinstances", {"_id":1}, 0)
```

Inside gov_api, property info cantain data to build up multipliers, you might need to parse it cause it is string. Inside info dict are 2 keys, tax and pf. pf means police fund.

tax contains list of taxes, and at the same time bonuses.

```json=
{
    "income_tax" : info_string['tax'][0],    # tax based on SIM price
    "police_tax" : info_string['tax'][1],    # tax to fund police
    "edu_tax" : info_string['tax'][2],       # tax to fund education
    "art_tax" : info_string['tax'][3],       # tax to fund creativity
    "war_tax" : info_string['tax'][4],       # tax to fund war
    "eco_tax" : info_string['tax'][5],       # tax for eco program
    "job_tax" : info_string['tax'][6],       # tax for jobs program
    "basic_tax" : info_string['tax'][7],     # tax for nothing, basic tax
    "rich_tax" : info_string['tax'][9],      # tax on top 3 or 5 players
    }
```
**total tax**:
```py
tax = sum(info_string['tax']) - info_string['tax'][9]
```
**1 :** ``"income_tax" : info_string['tax'][0],    # tax based on SIM price``
This one os based on average SIM token price in last 3 days, when price goes down, this tax should go up and decrease inflation.

**2 :**``"police_tax" : info_string['tax'][1],    # tax to fund police``

President by this tax funding police, and it works 4x better than without this tax:
```py
if info_string['tax'][1] == 1:
    police_multiplier = 1
else:
    police_multiplier = 0.25
```

Now you can calculate police station effect on crime:

```py
tech_police = 0
if "Police Equipment" in summary: # 1-1
    tech_police += 1
if "RoboCop" in summary: #1-3
    tech_police += 2

police_multiplier = info_string['pf']/100
if 'Police Station' in summary:
    police_effect = summary['Police Station'] * (0-(police_multiplier*(tech_police+2)))
else:
    police_effect = 0
```

**REMINDER** chance for crime is the only secret thing in game, here above you have just police station effect on crime.

**3 :** ``"edu_tax" : info_string['tax'][2],       # tax to fund education``

President can boost education by 10% and it cost players 5% of daily income

**4 :** ``"art_tax" : info_string['tax'][3],       # tax to fund creativity``

President can boost creativity by 10% and it cost players 5% of daily income

**5 :** ``"war_tax" : info_string['tax'][4],       # tax to fund war``

When president decide about war, players pay 10% tax and 80% of that goes to Military Industrial Complex owners

**6 :** ``"eco_tax" : info_string['tax'][5],       # tax for eco program``

For 5% tax president can boost income from Wind Turbines and Solar Panels

**7 :** ``"job_tax" : info_string['tax'][6],       # tax for jobs program``

For 5% tax president can double Job Agency effect

**8 :** ``"basic_tax" : info_string['tax'][7],     # tax for nothing, basic tax``

Basic tax from 1-20%. President can use it to reduce SIM inflation.


## 3.6 Production BEER 

1% from game event wallet of BEER is distributed back to players on daily basis.
Fully Automated Brewery technology doubles player shares in BEER production.

How to calculate BEER production for today:

```py
def beer_distribution(beer_distro):
    print("BEER DISTRIBUTION")
    ### BEER BALANCE
    beer_balance = engine_everything("tokens", "balances", {"account":"dcityevent", "symbol":"BEER"}, 0)
    ### TOTAL SHARES
    beer_shares = sum(beer_distro.values()) 
    for key in beer_distro:
        print("USER:", key, "SHARES:", beer_distro[key], "DISTRIBUTION:", round(0.01*float(beer_balance[0]['balance'])*beer_distro[key]/beer_shares,3))

def breweries():
    grouping = {}
    all_breweries = engine_everything("nft", "CITYinstances", {"properties.name":"Brewery"}, 0)
    for item in all_breweries:
        if item['account'] in grouping:
            grouping[item['account']] += 1
        else:
            grouping[item['account']] = 1

    ## Double shares on Technology
    for key in grouping:
        player_tech = engine_everything("nft", "CITYinstances", {"account": key, "properties.type":"tech"}, 0)
        for item in player_tech:
            if item['properties']['name'] == "Fully Automated Brewery":
                new_value = grouping[key]*2
                grouping[key] = new_value
                break
    
    return grouping

beer_distro = breweries()
beer_distribution(beer_distro)
```

## 3.7 Production WEED

1% from game event wallet of WEED is distributed back to players on daily basis.
WEED Fest doubles player shares in production.

How to calculate WEED distribution for today:

```py
def weed_distribution(weed_distro):
    print("WEED DISTRIBUTION")
    ### WEED BALANCE
    weed_balance = engine_everything("tokens", "balances", {"account":"dcityevent", "symbol":"WEED"}, 0)
    ### TOTAL SHARES
    weed_shares = sum(weed_distro.values()) 
    for key in weed_distro:
        print("USER:", key, "SHARES:", weed_distro[key], "DISTRIBUTION:", round(0.01*float(weed_balance[0]['balance'])*weed_distro[key]/weed_shares,3))

def weed_farms():
    grouping = {}
    farms = engine_everything("nft", "CITYinstances", {"properties.name":"WEED Farm"}, 0)
    for item in farms:
        if item['account'] in grouping:
            grouping[item['account']] += 1
        else:
            grouping[item['account']] = 1

    ## Double shares on WEED Fest
    for key in grouping:
        player_api = engine_everything("nft", "APIinstances", {"properties.x":key}, 0)
        try:
            e2 = ast.literal_eval(player_api[0]['properties']['e2'])
            if e2[4][1] > time.time()-(24*60*60):
                new_value = grouping[key] * 2
                grouping[key] = new_value
        except:
            print("no events")
    
    return grouping

weed_distro = weed_farms()  ## CHECK WHO HAVE WEED Farm and WEED Fest
weed_distribution(weed_distro) ## Dsitribution
```


## 3.8 Backgrounds

Backgrounds are minable NFT, players can trade it.
4 Groups of backgrounds:

Popular (60%): 
* Night
* Clear Sky
* Rockies
* Mountains

Normal (30%):
* Lake
* Snow
* Everest

Rare (9%):
* Asia
* Egypt

Ultra-Rare (1%):
* Atlantis

For list of backgrounds for account:
```py
my_bg = engine_everything("nft", "CITYinstances", {"account":username, "properties.type":"background"}, 0)
```

## 3.9 Chance for background

```py
if 'Art Gallery' in summary:
    bg_chance = round(min(25,creativity/80))
else:
    bg_chance = 0
```

## 3.10 dCity Logs

Memos in transactions sent to dcitylogs from 2 accounts, dcityassets and dcitymining.
First one for random cards from game second for citizens, technologies, backgrounds mined in game.

https://he.dtools.dev/@dcitylogs   ## ALL
https://he.dtools.dev/@dcityassets ## random cards
https://he.dtools.dev/@dcitymining ## mined cards
https://he.dtools.dev/@dcitytraining ## training

https://accounts.hive-engine.com/accountHistory?account=dcitylogs
https://accounts.hive-engine.com/accountHistory?account=dcityassets
https://accounts.hive-engine.com/accountHistory?account=dcitymining
https://accounts.hive-engine.com/accountHistory?account=dcitytraining

## 3.11 Simple every 24h data for City
If you don't want to build whole math but just display basic stats refreshed every 24h:

From API NFT's:
```py
username = "gerber"
player_api = engine_everything("nft", "APIinstances", {"properties.x":username}, 0)
my_stats = ast.literal_eval(player_api[0]['properties']['info'])
```
From rewards:
```py
username = "gerber"
url = "https://accounts.hive-engine.com/accountHistory?account=dcitygame"
rewards_history = requests.request("GET", url).json()
for b in rewards_history:
    if b['to'] == username:
        my_stats = b['memo'].split(", crime")[0]
        # my_stats = b['memo'].split(".")[0] # for crime data too
        break
print(my_stats)

```

4 Interactions with Game
===

## 4.1 Buying cards from game

Players can buy from 1 to 10 cards in one transaction. Price for one card is 4 HIVE or 800 SIM tokens.

Transfer of 4-40 HIVE to **dcitycards**
Transfer of 800-8000 SIM tokens to **dcitycards**

memo: **new**
for new release

## 4.2 Organizing Events

Players organizing events by sending tokens to **dcityevent**
First you should check if player can organize event:

```py
username = "gerber"
player_api = engine_everything("nft", "APIinstances", {"properties.x":username}, 0)

e = ast.literal_eval(player_api[0]['properties']['e'])
e2 = ast.literal_eval(player_api[0]['properties']['e2'])

if e[1][1]+(72*60*60) < time.time():    ## event can be organized every 72 hours
    beerfest = True                     ##  BEER fest

if e[2][1]+(72*60*60) < time.time():    ## event can be organized every 72 hours
    science_convention = True           ## Science Convention

if e[3][1]+(72*60*60) < time.time():    ## event can be organized every 72 hours
    art_convention = True               ## Art convention

if e2[4][1]+(72*60*60) < time.time():   ## event can be organized every 72 hours
    weed_fest = True                    ## WEED Fest
```

Event costs:

BEER Fest - 5 BEER
Science Convention - 100 STEM
Art Convention - 250 CCC
WEED Fest - 75 WEED


## 4.3 Governments actions (voting, president decisions)

Check Registered candidates:
```py
cands = find_one({"_id":3}, "API")
sum_cands = {}
for key in cands[0]['properties']:
    short_cands = ast.literal_eval(cands[0]['properties'][key])
    sum_cands = {**sum_cands, **short_cands}
    
print(sum_cands)
```

Check actual president:
```py
gov_api = engine_everything("nft", "APIinstances", {"_id":1}, 0)
president = gov_api[0]['properties']['gov']
```

How to build custom_json for voting:
```json
id = "dcity"
json_data = {"action": "gov_vote", "data": "user_to_vote_on"}
```

How to build custom_json for president decisions

``"action": "gov_decision"``
``"data": "basic tax"``
posible decisions: "police tax", "edu tax", "art tax", "war tax", "eco tax", "job tax", "basic tax"
``"value": 0`` 1-20 for "basic tax" 0 for rest
```json
id = "dcity"
json_data = {"action": "gov_decision", "data": "basic tax", "value": 12}
```
```json
id = "dcity"
json_data = {"action": "gov_decision", "data": "edu tax", "value": 0}
```

![](https://i.imgur.com/whwytQK.png)


## 4.4 Unlocking Technologies

In **player_api** you have property **tech** (remember data in API is always as string):

```py
tech = {'t': 111, 1: [0, 0, 0, 0, 0], 2: [0, 0, 0, 0, 0], 3: [0, 0, 0, 0, 0], 4: [0, 0, 0, 0, 0]}
```

t - is time of last initial research and base for cooldown
After unlocking technology player have 36 hours or 24 hours(with Better Documentation Practice tech) cooldown before he can unlock another one.

Let's check if player have that tech:
```py
my_tech = engine_everything("nft", "CITYinstances", {"account":username, "properties.type":"tech"}, 0)
summary_tech = {}
for tech in my_tech:
    if tech['properties']['name'] in summary_tech:
        summary_tech[tech['properties']['name']] += 1
    else:
        summary_tech[tech['properties']['name']] = 1
        
if 'Better Documentation Practice' in summary_tech:
    tech_cd = 24
else:
    tech_cd = 36
```

If you already have all the data from making calculations it can be easier:
```py

if 'Better Documentation Practice' in summary:
    tech_cd = 24
else:
    tech_cd = 36
```

Now you can check if player can unlock next technology or how much time left before he can do it:

```py
player_api = engine_everything("nft", "APIinstances", {"properties.x":username}, 0)
tech_string = ast.literal_eval(player_api[0]['properties']['tech'])

if time.time() > tech_string['t']+(tech_cd*60*60):
    unlock = True
else:
    unlock = False
    seconds_left = tech_string['t']+(tech_cd*60*60) - time.time()
```

To Unlock technology send:
Tier 1 - 30 SIM 
Tier 2 - 60 SIM
Tier 3 - 150 SIM
Tier 4 - 500 SIM
Tier 5 - 1000 SIM

To unlock technology player sends SIM to **dcitygame** with memo: branch-tier for example 1-1, 2-3, 4-1 etc

Let's build some code:
```py
unlock_prices = {1:30, 2:60, 3:150, 4:500, 5:1000}

all_tech = [["Police Equipment" , "Drone Technology", "RoboCop", "Neural Network", "Voice to Skull"],
         ["Free Internet Connection", "Better Documentation Practice", "Open Source", "Free Education", "Advanced Training"],
         ["Basic Automation", "Fully Automated Brewery", "AI Technology", "Mining Operation - BTC", "Advanced Robotics"],
         ["GMO Farming", "ECO Energy", "Cold Fusion", "Dyson Sphere", "Advanced Recycling"]]

if unlock == True:
    ## CHECK FOR POSSIBILITIES
    del tech_string['t']
    for key in tech_string:
        to_unlock = tech_string[key].count(1)
        if to_unlock == 5:
            print("player %s unlocked all tech for %s branch" % (username, key))
        else:
            print(" player %s can unlock %s by sending %s to dcitygame with memo: %s" % (username, all_tech[key][to_unlock] , unlock_prices[to_unlock+1], (str(key)+"-"+str(to_unlock+1))))
```

```
player gerber unlocked all tech for 1 branch
player gerber unlocked all tech for 2 branch
player gerber can unlock Advanced Recycling by sending 1000 to dcitygame with memo: 3-5
player gerber unlocked all tech for 4 branch

```

## 4.5 All in one tech-tree

```py
import json
import requests
import time
import ast

def engine_everything(contract, table, query, offset):
    url = 'https://api.hive-engine.com/rpc/contracts'
    params = {'contract':contract, 'table':table, 'query':query, 'limit':1000, 'offset':offset, 'indexes':[]}
    j = {'jsonrpc':'2.0', 'id':1, 'method':'find', 'params':params}

    with requests.post(url, json=j) as r:
        data = r.json()
        result = data['result']
        if len(result) == 1000:
            result += engine_everything("nft", "CITYinstances", query, offset+1000)
    return result

def tech_check():
    summary_tech = {}
    for tech in my_tech:
        if tech['properties']['name'] in summary_tech:
            summary_tech[tech['properties']['name']] += 1
        else:
            summary_tech[tech['properties']['name']] = 1
        
    if 'Better Documentation Practice' in summary_tech:
        tech_cd = 24
    else:
        tech_cd = 36

    if time.time() > tech_string['t']+(tech_cd*60*60):
        unlock = True
    else:
        unlock = False
        seconds_left = tech_string['t']+(tech_cd*60*60) - time.time()

    unlock_prices = {1:30, 2:60, 3:150, 4:500, 5:1000}

    all_tech = [["Police Equipment" , "Drone Technology", "RoboCop", "Neural Network", "Voice to Skull"],
         ["Free Internet Connection", "Better Documentation Practice", "Open Source", "Free Education", "Advanced Training"],
         ["Basic Automation", "Fully Automated Brewery", "AI Technology", "Mining Operation", "Advanced Robotics"],
         ["GMO Farming", "ECO Energy", "Cold Fusion", "Dyson Sphere", "Advanced Recycling"]]

    if unlock == True:
    ## CHECK FOR POSSIBILITIES
        del tech_string['t']
        for key in tech_string:
            to_unlock = tech_string[key].count(1)
            if to_unlock == 5:
                print("player %s uncloked all tech for %s branch" % (username, key))
            else:
                print("player %s can unlock %s by sending %s to dcitygame with memo: %s" % (username, all_tech[key][to_unlock] , unlock_prices[to_unlock+1], (str(key)+"-"+str(to_unlock+1))))

    if unlock == False:
        print("Player %s needs to wait %s min" % (username, round(seconds_left/60)))

username = "gerber"

my_tech = engine_everything("nft", "CITYinstances", {"account":username, "properties.type":"tech"}, 0)
player_api = engine_everything("nft", "APIinstances", {"properties.x":username}, 0)

try:
    tech_string = ast.literal_eval(player_api[0]['properties']['tech'])
    tech_check()
except:
    print("tech property doesn't exist player %s can unlock all the tier one technologies by sending 30 SIM to dcitygame with memo 1-1, 2-1, 3-1, 4-1" % username)
```

## 4.6 Picking background

First you can check backgrounds for a player:
```py
my_bg = engine_everything("nft", "CITYinstances", {"account":username, "properties.type":"background"}, 0)
```

custom_json for picking background:
```json=
id = "dcity"
json_data = {"action": "pick_bg", "data": "picked_background"
```

5 Market
===

## 5.1 Open Orders
```py
open_orders = engine_everything("nftmarket", "CITYsellBook", {}, 0)
```
## 5.2 24h Trade Hsitory
```py
tradehistory24h = engine_everything("nftmarket", "CITYtradesHistory", {}, 0)
```
