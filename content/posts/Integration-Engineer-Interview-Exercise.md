---
layout: post
title: "Integration Engineer Interview Exercise"
categories: Write-Up
date: 2023-03-23
image: /images/data_engineer_interview_question/PandT.png
description: A solution for solving an API response parsing exercise given in an engineering interview.
tags: [Python, JavaScript]
katex: true
markup: "markdown"
---

![Image](/images/integration_engineer_interview_exercise/PandT.png#center)

******

## The Exercise

In a recent engineering interview I was given a somewhat dense API response JSON file and asked to write solutions in both Python and JavaScript/TypeScript for the following tasks:
> - *Load data from JSON file.*
> - *Output the domain with the highest risk score and the domain with the lowest risk score (If tied use the first occurrence).*
> - *What’s the average of all the domain risk scores.*
> - *Print a list of unique IP addresses.*
> - *Tell me all the domains which contains “phishing” as one of its threats.*

&nbsp;

By manually inspecting the JSON file I found that the information I needed to gather was structured in the following representation:

```json
{
  "response": {
    "results": [
      {
        "domain": "string",
        "domain_risk": {
          "risk_score": "number",
          "components": [
            {
              "name": "string"
            }
          ]
        },
        "ip": [
          {
            "address": {
              "value": "string"
            }
          }
        ]
      }
    ]
  }
}
```

******

## Python Solution

My Python solution code:
```
"""solution.py"""

import json
from typing import Tuple, List
from statistics import mean

FILE_NAME = "some_dt_data_from_investigate.json"

SolutionReturnType = Tuple[List[Tuple[int, str]], List[str], List[Tuple[str,
                                                                        str]]]


def solution(file_name: str) -> SolutionReturnType:
    """
    Function to parse investigation file, returning various
    information.

    Args:
        file_name (str): File name of investigation file.

    Returns:
        SolutionReturnType:
        A list containing three lists:
            1. A list of tuples, each containing a domain's risk
               score and name.
            2. A list of IP addresses extracted from the JSON data.
            3. A list of tuples, each containing a domain name and
               its associated phishing component.
    """

    # Load data from JSON file.
    with open(file_name, "r", encoding='utf-8') as data_file:
        data = json.load(data_file)

    # Initialize empty lists.
    _scores: List[Tuple[int, str]] = []
    _ips: List[str] = []
    _phishing: List[Tuple[str, str]] = []

    # Loop through results objects in JSON data.
    for result in data['response']['results']:

        # Append a tuple containing the domain's risk score and name to
        #   _scores list.
        _scores.append((result['domain_risk']['risk_score'], result['domain']))

        # Append each IP address to _ips list.
        for addresses in result['ip']:
            _ips.append(addresses['address']['value'])

        # Append a Tuple containing the domain name and its phishing component
        #   to _phishing list.
        for component in result['domain_risk']['components']:
            if 'phishing' in component['name']:
                _phishing.append((result['domain'], component['name']))

    # Return a list containing the _scores, _ips, and _phishing lists.
    return (_scores, _ips, _phishing)


if __name__ == "__main__":
    scores, ips, phishing = solution(FILE_NAME)

    # 1 #
    # Output the domain with the highest risk score and the domain with the
    #     lowest risk score (If tied use the first occurrence).
    print(max(scores, key=lambda x: x[0]))
    print(min(scores, key=lambda x: x[0]))

    # 2 #
    # What’s the average of all the domain risk scores.
    print(mean([i[0] for i in scores]))

    # 3 #
    # Print a list of unique IP addresses.
    print(set(ips))

    # 4 #
    # Tell me all the domains which contains “phishing” as one of its threats.
    print(phishing)
```

Output:
```bash
(100, 'bsvclaim.com')
(15, 'hellcase.su')
76.93333333333334
{'91.195.240.13', '194.58.56.8', '194.67.71.88', '194.58.56.118',
'51.161.21.1', '172.67.208.170', '194.58.56.137', '194.67.71.81',
'104.21.93.103', '23.254.229.115', '194.67.71.165', '209.99.40.225',
'194.67.71.166', '194.67.71.128', '35.209.152.88', '194.67.71.124',
'194.67.71.175', '107.164.44.64', '194.58.56.191', '194.58.56.67',
'194.58.56.80', '66.23.232.40', '194.67.71.67', '194.58.56.131',
'172.67.170.253', '194.67.71.127', '194.67.71.43', '194.58.56.190',
'194.58.56.88', '194.58.56.211', '194.67.71.8', '85.119.149.127',
'194.67.71.125', '194.58.56.13', '87.236.16.223', '104.148.37.236',
'194.67.71.106', '194.58.56.171', '52.128.23.153', '172.67.175.60',
'194.67.71.194', '194.67.71.15', '194.67.71.157', '194.58.56.78',
'194.58.56.147', '194.58.56.29', '194.58.56.107', '69.64.147.28',
'81.91.170.22', '194.67.71.23', '34.98.99.30', '194.67.71.73',
'198.54.117.244', '194.58.56.74', '194.58.56.40', '192.64.117.42',
'194.58.56.28', '194.58.56.155', '104.21.31.66', '104.21.87.223',
'91.195.240.117', '194.67.71.99', '194.67.71.11'}
[('5000eth.net', 'threat_profile_phishing'),
('5000eth.su', 'threat_profile_phishing'),
('500btc.su', 'threat_profile_phishing'),
('bcvclaim.com', 'threat_profile_phishing'),
('binance5000.com', 'threat_profile_phishing'),
('binanceairdrop.net', 'threat_profile_phishing'),
('binancebonus.com', 'threat_profile_phishing'),
('binanceclaim.com', 'threat_profile_phishing'),
('binancedrop.com', 'threat_profile_phishing'),
('binanceprize.net', 'threat_profile_phishing'),
('binancepromo.net', 'threat_profile_phishing'),
('binancex10.com', 'threat_profile_phishing'),
('bitcoindrop.net', 'threat_profile_phishing'),
('bitcoinprize.net', 'threat_profile_phishing'),
('bsvclaim.com', 'threat_profile_phishing'),
('btc-drop.com', 'threat_profile_phishing'),
('btcbinance.com', 'threat_profile_phishing'),
('btcdrop.net', 'threat_profile_phishing'),
('btcevent.net', 'threat_profile_phishing'),
('btcholiday.net', 'threat_profile_phishing'),
('buterinpromo.com', 'threat_profile_phishing'),
('claim-tool.com', 'threat_profile_phishing'),
('csgo-exchnge.online', 'threat_profile_phishing'),
('csgo1x1.com', 'threat_profile_phishing'),
('csgo24.trade', 'threat_profile_phishing'),
('csgo2bets.com', 'threat_profile_phishing'),
('csgocompetive.com', 'threat_profile_phishing'),
('csgohightbet.com', 'threat_profile_phishing'),
('csgoswap.de', 'threat_profile_phishing'),
('csgotrade.money', 'threat_profile_phishing'),
('csgotrade.su', 'threat_profile_phishing'),
('csgowinx.com', 'threat_profile_phishing'),
('csgowix.com', 'threat_profile_phishing'),
('d2faceit.com', 'threat_profile_phishing'),
('eth-drop.com', 'threat_profile_phishing'),
('ethbuterin.com', 'threat_profile_phishing'),
('ethdrop.net', 'threat_profile_phishing'),
('ethprize.net', 'threat_profile_phishing'),
('ethpromo.net', 'threat_profile_phishing'),
('fast-trade24.com', 'threat_profile_phishing'),
('fastpromo.su', 'threat_profile_phishing'),
('skinstrade.zone', 'threat_profile_phishing'),
('skinstradefast.su', 'threat_profile_phishing'),
('tradeskinsfast.su', 'threat_profile_phishing')]
```

******

## TypeScript Solution

My TypeScript solution code:
```ts
// solution.ts

const fileName: string = './some_dt_data_from_investigate.json';

interface InvestigationData {
  response: {
    results: {
      domain: string;
      domain_risk: {
        risk_score: number;
        components: {
          name: string;
        }[];
      };
      ip: {
        address: {
          value: string;
        };
      }[];
    }[];
  };
}

type solutionReturnType = [
  { [key: number]: string }[],
  string[],
  { [key: string]: string }
];


/**
 * Function to parse investigation file, returning various information.
 *
 * @param {string} jsonDataFile - Name of JSON file containing investigation data.
 *
 * @returns {solutionReturnType} - An array containing three arrays:
 *   1. An array of objects, each containing a domain's risk score and name.
 *   2. An array of IP addresses extracted from the JSON data.
 *   3. An object containing a domain name and its associated phishing component.
 */
 function solution(jsonDataFile: string): solutionReturnType {

  // Load data from JSON file.
   const data: InvestigationData = require(jsonDataFile);

   // Initialize empty arrays.
   const _scores: { [key: number]: string }[] = [];
   const _ips: string[] = [];
   const _phishing: { [key: string]: string } = {};

  // Loop through results objects in JSON Object.
  for (let result of data.response.results) {

    // Push an object containing the domain's risk score and name to _scores array.
    const domainScore = result.domain_risk.risk_score;
    const domainName = result.domain;
    _scores.push({[result.domain_risk.risk_score]: result.domain});

    // Push each IP address to _ips array.
    for (let addresses of result.ip) {
      _ips.push(addresses.address.value);
    }

    // Push an object containing the domain name and its phishing component to _phishing array.
    for (let component of result.domain_risk.components) {
      if (component.name.includes('phishing')) {
        _phishing[result.domain] = component.name;
      }
    }
  }

  // Return an array containing the _scores, _ips, and _phishing arrays.
  return [_scores, _ips, _phishing];
}


const [scores, ips, phishing] = solution(fileName);

// # 1 Output the domain with the highest risk score and the domain with the
//      lowest risk score (If tied the first occurrence).
let minKey: number | null = null;
let maxKey: number | null = null;
scores.reduce((_, obj) => {
  const key = parseInt(Object.keys(obj)[0]);
  if (maxKey === null || key > maxKey) {
    maxKey = key;
  }
  if (minKey === null || key < minKey) {
    minKey = key;
  }
  return null;
}, null);
console.log(scores.filter((obj) => parseInt(Object.keys(obj)[0]) === maxKey)[0]);
console.log(scores.filter((obj) => parseInt(Object.keys(obj)[0]) === minKey)[0]);

// # 2 What’s the average of all the domain risk scores
const nums = scores.map(obj => parseInt(Object.keys(obj)[0]));
const sum = nums.reduce((acc, curr) => acc + curr);
console.log(sum / nums.length);

// # 3 Print a list of unique IP addresses.
console.log([... new Set(ips)]);

// # 4 Tell me all the domains which contains “phishing” as one of its threats.
console.log(phishing);
```

Output:
```bash
{ '100': 'bsvclaim.com' }
{ '15': 'hellcase.su' }
76.93333333333334
[
  '194.67.71.127',  '194.67.71.8',    '194.58.56.191',
  '194.58.56.78',   '194.58.56.171',  '194.58.56.28',
  '194.67.71.15',   '194.67.71.165',  '194.67.71.67',
  '194.67.71.81',   '194.67.71.128',  '194.67.71.157',
  '91.195.240.13',  '35.209.152.88',  '52.128.23.153',
  '104.21.87.223',  '172.67.170.253', '194.67.71.11',
  '194.67.71.125',  '104.21.31.66',   '172.67.175.60',
  '51.161.21.1',    '107.164.44.64',  '194.67.71.124',
  '194.67.71.43',   '23.254.229.115', '34.98.99.30',
  '85.119.149.127', '192.64.117.42',  '194.67.71.106',
  '194.67.71.166',  '194.67.71.175',  '194.67.71.194',
  '194.67.71.88',   '194.67.71.99',   '194.58.56.137',
  '194.58.56.80',   '194.58.56.88',   '91.195.240.117',
  '194.58.56.190',  '194.58.56.118',  '194.58.56.147',
  '194.58.56.107',  '81.91.170.22',   '69.64.147.28',
  '194.58.56.29',   '194.58.56.13',   '194.58.56.40',
  '209.99.40.225',  '104.148.37.236', '104.21.93.103',
  '172.67.208.170', '66.23.232.40',   '87.236.16.223',
  '198.54.117.244', '194.58.56.155',  '194.58.56.131',
  '194.58.56.8',    '194.67.71.23',   '194.67.71.73',
  '194.58.56.211',  '194.58.56.67',   '194.58.56.74'
]
{
  '5000eth.net': 'threat_profile_phishing',
  '5000eth.su': 'threat_profile_phishing',
  '500btc.su': 'threat_profile_phishing',
  'bcvclaim.com': 'threat_profile_phishing',
  'binance5000.com': 'threat_profile_phishing',
  'binanceairdrop.net': 'threat_profile_phishing',
  'binancebonus.com': 'threat_profile_phishing',
  'binanceclaim.com': 'threat_profile_phishing',
  'binancedrop.com': 'threat_profile_phishing',
  'binanceprize.net': 'threat_profile_phishing',
  'binancepromo.net': 'threat_profile_phishing',
  'binancex10.com': 'threat_profile_phishing',
  'bitcoindrop.net': 'threat_profile_phishing',
  'bitcoinprize.net': 'threat_profile_phishing',
  'bsvclaim.com': 'threat_profile_phishing',
  'btc-drop.com': 'threat_profile_phishing',
  'btcbinance.com': 'threat_profile_phishing',
  'btcdrop.net': 'threat_profile_phishing',
  'btcevent.net': 'threat_profile_phishing',
  'btcholiday.net': 'threat_profile_phishing',
  'buterinpromo.com': 'threat_profile_phishing',
  'claim-tool.com': 'threat_profile_phishing',
  'csgo-exchnge.online': 'threat_profile_phishing',
  'csgo1x1.com': 'threat_profile_phishing',
  'csgo24.trade': 'threat_profile_phishing',
  'csgo2bets.com': 'threat_profile_phishing',
  'csgocompetive.com': 'threat_profile_phishing',
  'csgohightbet.com': 'threat_profile_phishing',
  'csgoswap.de': 'threat_profile_phishing',
  'csgotrade.money': 'threat_profile_phishing',
  'csgotrade.su': 'threat_profile_phishing',
  'csgowinx.com': 'threat_profile_phishing',
  'csgowix.com': 'threat_profile_phishing',
  'd2faceit.com': 'threat_profile_phishing',
  'eth-drop.com': 'threat_profile_phishing',
  'ethbuterin.com': 'threat_profile_phishing',
  'ethdrop.net': 'threat_profile_phishing',
  'ethprize.net': 'threat_profile_phishing',
  'ethpromo.net': 'threat_profile_phishing',
  'fast-trade24.com': 'threat_profile_phishing',
  'fastpromo.su': 'threat_profile_phishing',
  'skinstrade.zone': 'threat_profile_phishing',
  'skinstradefast.su': 'threat_profile_phishing',
  'tradeskinsfast.su': 'threat_profile_phishing'
}
```


******

## Conclusion

While not a particularly difficult exercise, the JSON file was fairly dense and it did seem very realistic and relevant considering the role that I was interviewing for. I was not able to complete the exercise within the designated time frame in the interview due to nervousness, but easily completed it after I had flunked the technical interview.


******


