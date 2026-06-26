# TryHackMe – WatterBottle Writeup

## Overview

Today we are solving another **TryHackMe Easy OSINT challenge** named **WatterBottle**.

Unlike traditional penetration testing machines, this challenge focuses entirely on **Open-Source Intelligence (OSINT)**. Instead of exploiting a vulnerable service, our objective is to investigate publicly available information and reconstruct details about a business that existed in the past.

The scenario provides only a few clues:

* A former water refilling station located near **Boni Avenue**
* The station existed around **2014**
* Its contact number contains **12 digits**
* The number starts with **63922**
* The station has since been replaced by another business

Our task is to identify both:

* The original water station name
* Its original contact number

The final answer must follow the format:

```text
THM{stationname_contactnumber}
```

---

# Scenario

The challenge presents the following situation:

A person returned to their hometown and wanted to refill water from a station they frequently visited until **2014**. Unfortunately, they no longer remembered the station's name or phone number.

The only details remembered were:

* The station was located near **Boni Avenue**
* It existed before 2014
* It has now been replaced by another establishment
* The contact number began with:

```text
63922
```

---

# Initial Investigation

The first step was identifying the location.

Searching for **Boni Avenue** revealed that it is located in the **Philippines**, specifically within Metro Manila.

Since the challenge specifically references an old business, the investigation shifted toward finding historical business information rather than current businesses.

Searches included combinations of:

* Boni Avenue water station
* Water refilling station Boni Avenue
* Boni Avenue water refill 2014
* Historical business listings
* Archived contact information

---

# Historical Business Research

While researching water refilling stations operating around Boni Avenue during **2014**, multiple businesses appeared.

One of the names encountered during the investigation was:

```text
Aqua Safe
```

At first glance this looked promising, but deeper investigation showed that this was not the original company being referenced.

Continuing the research eventually led to **Aquabest**, which operated water refilling stations in the area during the relevant time period.

This matched the historical timeline much better than newer businesses currently present near Boni Avenue.

---

# Contact Information Investigation

After identifying the likely company, the next objective was locating its historical contact number.

The official company contact page was reviewed during the investigation.

From the publicly available company information, a contact number beginning with the expected prefix was identified.

During further verification and historical cross-referencing, the original Boni Avenue branch contact was determined to be:

```text
639228721288
```

This matched the challenge requirements:

* 12-digit number
* Begins with **63922**

---

# Station Identified

The original water station was identified as:

```text
Aquabest
```

---

# Contact Number

The historical contact number associated with the station was determined to be:

```text
639228721288
```

---

# Flag Construction

The challenge requires:

* Water station name in lowercase
* Underscore separator
* Contact number

Therefore:

```text
THM{aquabest_639228721288}
```

---

# Methodology Summary

### Step 1

Identify the geographical location mentioned in the scenario.

* Boni Avenue
* Metro Manila
* Philippines

---

### Step 2

Research historical water refilling stations operating near Boni Avenue around **2014**.

---

### Step 3

Compare historical business records rather than relying only on current map listings, since the original business had already been replaced.

---

### Step 4

Investigate official company information and archived references to identify historical contact numbers.

---

### Step 5

Verify that the recovered phone number satisfies the challenge conditions:

* Begins with **63922**
* Contains **12 digits**

---

### Step 6

Construct the flag using the required format.

---

# Final Answer

## Water Station

```text
Aquabest
```

## Contact Number

```text
639228721288
```

## Flag

```text
THM{aquabest_639228721288}
```

---

# Conclusion

This challenge demonstrates a simple but effective OSINT workflow using publicly available information. Instead of exploiting systems, the investigation relied on historical business records, official company information, location-based searches, and careful verification of contact details. By correlating the location, timeline, and phone number pattern provided in the scenario, the original water station was successfully identified as **Aquabest**, along with its corresponding historical contact number, allowing the correct flag to be constructed.
