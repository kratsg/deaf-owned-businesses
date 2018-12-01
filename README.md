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

  var response = UrlFetchApp.fetch(ghURI + "/git/refs?access_token="+ghToken, options);
}

function create_business_file(branch_name, biz_obj, timestamp, name, email){
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

  var filename = Utilities.formatDate(new Date("11/30/2018 14:56:16"), "GMT", "YY-MM-dd") + "-" + sanitize_string(biz_obj['name'].replace(/[\s]+/g,'-'));
  var response = UrlFetchApp.fetch(ghURI + "/contents/_data/businesses/"+filename+".json?access_token="+ghToken, options);
}

function create_pull_request(branch_name, biz_obj){
  var body = "```\n" + JSON.stringify(biz_obj, null, 4) +
             "```\n\n" +
             "Submitted by [" + name + "](mailto:" + email + ") at " + timestamp + " from [DeafBiz Google Form](https://docs.google.com/forms/d/e/1FAIpQLSdFPZyanfu-_nkWpKKwIIk15NzeoBXgkg5xSp4c5CrBlSoARw/viewform) automatically.";

  var payload = {
    "title": "Add "+biz_obj['name'],
    "body": "Please add " + biz_obj['name'] + " to the list of businesses. Thanks.\n\n" + body,
    "head": branch_name,
    "base": "master"
  };
  var options = {
    "method": "POST",
    "contentType": "application/json",
    "payload": JSON.stringify(payload),
    "headers": {"Accept": "application/vnd.github.v3+json"}
  };

  var response = UrlFetchApp.fetch(ghURI + "/pulls?access_token="+ghToken, options);
}

function add_new_business(biz_obj, timestamp, name, email){
  var branch_name = "add/business/"+sanitize_string(biz_obj['name'])+"/"+sanitize_string(timestamp);
  create_branch(branch_name);
  create_business_file(branch_name, biz_obj, timestamp, name, email);
  create_pull_request(branch_name, biz_obj);
}

function onFormSubmit(e) {

  var timestamp = e.values[0];
  var email = e.values[1];
  var name = e.values[2];

  var bizname = e.values[3];
  var url = e.values[4];
  var location = (e.values[5] == "")?"null":e.values[5];
  var comments = (e.values[6] == "")?"null":e.values[6];
  var category = (e.values[7] == "")?"null":e.values[7];

  var biz_obj = {"category": category,
                 "comments": comments,
                 "location": location,
                 "name": bizname,
                 "url": url};

  add_new_business(biz_obj, timestamp, name, email);
}
```
