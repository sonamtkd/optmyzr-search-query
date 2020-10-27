/***************************************************
* Undefined Search Terms Report
* @version 1.1
* @author: Naman Jindal (Optmyzr)
****************************************************/

var LAST_N_DAYS = 30; // Number of previous days to include in report
var EMAILS = ['example@example.com']; // Array of Emails to be notified and given access to the results in a Google Sheet
var PRIMARY_METRIC = 'Cost'; // E.g. Impressions, Cost, Clicks

function main() {
  var map = {};
  var DATE_RANGE = getAdWordsFormattedDate(LAST_N_DAYS, 'yyyyMMdd') + ',' + getAdWordsFormattedDate(1, 'yyyyMMdd');
  var query = [
    'SELECT Date, Impressions, Cost, Clicks FROM ACCOUNT_PERFORMANCE_REPORT',
    'WHERE AdNetworkType1 = SEARCH', 'DURING', DATE_RANGE
  ].join(' ');
  
  var rows = AdsApp.report(query).rows();
  while(rows.hasNext()) {
    var row = rows.next();
    map[row.Date] = {
      'ACTUAL': 0,
      'QUERIES': 0
    };
    map[row.Date].ACTUAL = parseInt(row[PRIMARY_METRIC], 10);
  }
  
  var query = [
    'SELECT Date, Query, Impressions, Cost, Clicks FROM SEARCH_QUERY_PERFORMANCE_REPORT',
    'WHERE AdNetworkType1 = SEARCH',
    'DURING', DATE_RANGE
  ].join(' ');
  
  var rows = AdsApp.report(query).rows();
  while(rows.hasNext()) {
    var row = rows.next();
    map[row.Date].QUERIES += parseInt(row[PRIMARY_METRIC], 10);
  }
  
  var output = [];
  for(var date in map) {
    output.push([date, map[date].ACTUAL, map[date].QUERIES, (map[date].ACTUAL - map[date].QUERIES) / map[date].ACTUAL]);
  }
  
  if(!output.length) {
    Logger.log('No data in the account'); 
  }
  
  var TEMPLATE_URL = 'https://docs.google.com/spreadsheets/d/1G1-zPqm0kqQjZSPEwS8cfndYVzkswbyl80SiLlqiPF8/edit#gid=0';
  var template = SpreadsheetApp.openByUrl(TEMPLATE_URL);
  var ss = template.copy(AdsApp.currentAccount().getName() + ' - Undefined Search Terms Report by ' + PRIMARY_METRIC);
  ss.addEditors(EMAILS);
  
  var tab = ss.getSheets()[0];
  tab.getRange(2,1,tab.getLastRow(),tab.getLastColumn()).clearContent();
  tab.getRange(2,1,output.length,output[0].length).setValues(output).sort([{'column': 1, 'ascending': true}]); 
  
  var msg = 'Hi,\nPlease find below the undefined search terms report for your Google Ads account:\n'+ss.getUrl();
  MailApp.sendEmail(EMAILS.join(','), AdsApp.currentAccount().getName() + ' - Undefined Search Terms Report by ' + PRIMARY_METRIC, msg); 
  Logger.log("Your report is ready at " + ss.getUrl());
}


function round_(num,n) {    
  return +(Math.round(num + "e+"+n)  + "e-"+n);
}

function getAdWordsFormattedDate(d, format){
  var date = new Date();
  date.setDate(date.getDate() - d);
  return Utilities.formatDate(date,AdsApp.currentAccount().getTimeZone(),format);
}
