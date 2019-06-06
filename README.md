# Submitting a business

To submit a new business, edit [_data/businesses.yml](_data/businesses.yml) to add the new entry. Then submit a [pull request](https://github.com/kratsg/pulls) to get it on the website.

Feel free to file an issue instead of a pull request.

# Google Forms Automation

The [google form](https://docs.google.com/forms/d/e/1FAIpQLSdFPZyanfu-_nkWpKKwIIk15NzeoBXgkg5xSp4c5CrBlSoARw/viewform) has a script to automatically create a pull request for the new businesses. This script is:

```javascript
var ghToken = "PERSONAL_ACCESS_TOKEN";
var ghURI = "https://api.github.com/repos/kratsg/deaf-owned-businesses";
    
function sanitize_string(input){
  return input.replace(/[^\w_-]+/g,"").toLowerCase();
}

function get_master_sha1(){
  var options = {
    "method": "GET",
    "contentType": "application/json",
    "headers": {"Accept": "application/vnd.github.v3+json"}
  };

  var response = UrlFetchApp.fetch(ghURI + "/git/refs/heads/master?access_token="+ghToken, options);
  var json = response.getContentText();
  var data = JSON.parse(json);
  return data['object']['sha'];  
}

function create_branch(branch_name){
  var payload = {
    "ref": "refs/heads/" + branch_name,
    "sha": get_master_sha1()
  };
  var options = {
    "method": "POST",
    "contentType": "application/json",
    "payload": JSON.stringify(payload),
    "headers": {"Accept": "application/vnd.github.v3+json"}
  };
  console.log({'function': 'create_branch', 'payload': payload, 'options': options});
  var response = UrlFetchApp.fetch(ghURI + "/git/refs?access_token="+ghToken, options);
}

function update_business_file(filename, branch_name, biz_obj, sha, timestamp, name, email){
  var payload = {
    "message": "update existing business " + biz_obj['name'],
    "committer": {
      "name": name,
      "email": email
    },
    "branch": branch_name,
    "content": Utilities.base64Encode(JSON.stringify(biz_obj, null, 4), Utilities.Charset.UTF_8),
    "sha": sha
  };
  var options = {
    "method": "PUT",
    "contentType": "application/json",
    "payload": JSON.stringify(payload),
    "headers": {"Accept": "application/vnd.github.v3+json"}
  };
  console.log({'function': 'update_business_file', 'payload': payload, 'options': options});
  var response = UrlFetchApp.fetch(ghURI + "/contents/_data/businesses/"+filename+".json?access_token="+ghToken, options);
}


function create_business_file(filename, branch_name, biz_obj, timestamp, name, email){
  var payload = {
    "message": "add new business " + biz_obj['name'],
    "committer": {
      "name": name,
      "email": email
    },
    "branch": branch_name,
    "content": Utilities.base64Encode(JSON.stringify(biz_obj, null, 4), Utilities.Charset.UTF_8)
  };

  var options = {
    "method": "PUT",
    "contentType": "application/json",
    "payload": JSON.stringify(payload),
    "headers": {"Accept": "application/vnd.github.v3+json"}
  };
  console.log({'function': 'create_business_file', 'payload': payload, 'options': options});
  var response = UrlFetchApp.fetch(ghURI + "/contents/_data/businesses/"+filename+".json?access_token="+ghToken, options);  
}

function create_or_update_business_file(branch_name, biz_obj, timestamp, name, email){
  var options = {
    "method": "GET",
    "contentType": "application/json",
    "headers": {"Accept": "application/vnd.github.v3+json"},
    "muteHttpExceptions": true
  };
  var filename = sanitize_string(biz_obj['name'].replace(/[\s]+/g,'-'));
  var response = UrlFetchApp.fetch(ghURI + "/contents/_data/businesses/"+filename+".json?access_token="+ghToken, options);
  
  // file doesn't exist, create it
  if(response.getResponseCode() == 404){
    create_business_file(filename, branch_name, biz_obj, timestamp, name, email);
    return "add";
  }
  // file exists, update it
  else if(response.getResponseCode() == 200){
    var json = response.getContentText();
    var data = JSON.parse(json);
    update_business_file(filename, branch_name, biz_obj, data['sha'], timestamp, name, email);
    return "update";
  }
}

function create_pull_request(branch_name, biz_obj, add_or_update, timestamp, name, email){
  var body = "Please " + add_or_update + " " + biz_obj['name'] + ". Thanks.\n\n" + 
             "Submitted by [" + name + "](mailto:" + email + ") at " + timestamp + " from [DeafBiz Google Form](https://docs.google.com/forms/d/e/1FAIpQLSdFPZyanfu-_nkWpKKwIIk15NzeoBXgkg5xSp4c5CrBlSoARw/viewform) automatically.";

  var payload = {
    "title": add_or_update.charAt(0).toUpperCase() + add_or_update.slice(1) + " " + biz_obj['name'],
    "body": body,
    "head": branch_name,
    "base": "master"
  };
  var options = {
    "method": "POST",
    "contentType": "application/json",
    "payload": JSON.stringify(payload),
    "headers": {"Accept": "application/vnd.github.v3+json"}
  };
  console.log({'function': 'create_pull_request', 'payload': payload, 'options': options});
  var response = UrlFetchApp.fetch(ghURI + "/pulls?access_token="+ghToken, options);  
}

function add_or_update_business(biz_obj, timestamp, name, email){
  var branch_name = "add/business/"+sanitize_string(biz_obj['name'])+"/"+sanitize_string(timestamp);
  console.log({'function': 'add_or_update_business', 'branch_name': branch_name});
  create_branch(branch_name);
  var add_or_update = create_or_update_business_file(branch_name, biz_obj, timestamp, name, email);
  console.log({'function': 'add_or_update_business', 'add_or_update': add_or_update});
  create_pull_request(branch_name, biz_obj, add_or_update, timestamp, name, email);
}

// hit by multiple submission bug here: https://issuetracker.google.com/issues/124283798
function onFormSubmit(e) {
  console.log({'e.range.column': [e.range.columnStart, e.range.columnEnd]});
  if (e.range.columnStart == e.range.columnEnd) return;
  console.log('onFormSubmit: did not skip');  
  var timestamp = e.values[0];
  var email = e.values[1];
  var name = e.values[2];

  if(name == "" || email == "") return;
  
  var bizname = e.values[3];
  var url = e.values[4];
  var location = (e.values[5] == "")?"null":e.values[5];
  var comments = (e.values[6] == "")?"null":e.values[6];
  var category = (e.values[7] == "")?"null":e.values[7];

  var biz_obj = {"category": category,
                 "comments": comments,
                 "location": location,
                 "name": bizname,
                 "url": url,
                 "lastUpdated": timestamp};
  console.log({'function': 'onFormSubmit', 'e': e.values, 'biz_obj': biz_obj});
  add_or_update_business(biz_obj, timestamp, name, email);
}
```
