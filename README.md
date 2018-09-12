# AccountsIQ-POS integration (C#)
Processing CSVs from POS multiple systems then posting to accounting ledgers using API

### Background
I was asked to developed a system to somehow to bridge the gap the between POS system at chain hotels and the cloud-based enterprise accounting system at office of the franchise office. At the time, reports where being sent to the franchise office where the accountant compiled a spreadsheet using the summary dollar amounts from the hotel report. Once the transaction amounts balanced they were manually posted to the accounting system (AccountsIQ).  This manual system was being done for each day for about a dozen hotels. 

After researching the situation, the only data available was the CSV files that could be exported from each hotel. They could be exported each day and e-mailed to the office of the franchise holder.

### Solution
I developed a Windows C# desktop application for use by the accountant. The account just had to designate a folder on his computer and drop all the incoming CSV files in there. My application would take it from there and delve into each file to determine its validity, then extract and display information to the end user to post. 
My program recognizes the files below. At first I was provide only the CSV files for report RVYACCTG. Later it was determined this report lacks needed data and recently CSV reports RECRCP and GSTSTAT were found that gives me complete data.

```
Here is the “3 Pack” of files needed for each day...
RVYACCTG - PTD/YTD Accounting Report
RECRCP - Receivable Recap
GSTSTAT - Guest Statistics
```

The screen below shows how CSV file information is presented. Files are grouped by report date and location with the most recent dates at the top. Groups of files that are valid “3 packs” are denoted by alternating yellow and blue colors. The user can select any one of the valid files, click on Import to process transactions from the three files for that location/date. (A) Designates the folder for CSV files which can be changed on demand. (B) displays CSV files found in the folder. C Lines of data in each file. In parenthesis is the number of dollar amounts retrieved from each file. Zero dollar amounts are ignored. D Details of what tasks the program is doing in the background. 

![impscrn_01_b](https://user-images.githubusercontent.com/23184069/45405782-dfd51c00-b629-11e8-81ce-cf9f3ba43707.jpg)

```
```
This screen is just the sceen above that's been scrolled down. It shows CVS files where the "3 pack" is not complete and CSV files that are not recognized at all. 
![impscrn_02_b](https://user-images.githubusercontent.com/23184069/45405829-04c98f00-b62a-11e8-860b-0497d086319c.jpg)

```
```
The screen below shows data aggregated from a valid 3-pack. It is here the user assigns GL accounts for each dollar amount for posting to the accounting ledgers. This process maybe tedious at first but the program remembers the user's preference for each amount description from the CSV file. After a few days, probably all the accounts will be mapped. Each line has a note in brackets at the end. This the report code and line number for each amount should the user want to peer into the original CSV file to have a look at the original data. A key feature is the “suspense” option. The POS CSV files have a number of quirks including the fact the some dollar amounts are to the penny, but most are rounded to the dollar. Almost always you’ll have journal entries that don’t balance due to rounding errors. To account for this, my program allows the user to send the out-of-balance amount to a suspense account so it can post. 
![gl-si-sr-screen_b](https://user-images.githubusercontent.com/23184069/45406811-3db73300-b62d-11e8-980d-63ef541dafc8.jpg)
