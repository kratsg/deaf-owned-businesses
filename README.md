# Submitting a business

To submit a new business, edit [_data/businesses.yml](_data/businesses.yml) to add the new entry. Then submit a [pull request](https://github.com/kratsg/pulls) to get it on the website.

Feel free to file an issue instead of a pull request.

# Google Forms Automation

The [google form](https://docs.google.com/forms/d/e/1FAIpQLSdFPZyanfu-_nkWpKKwIIk15NzeoBXgkg5xSp4c5CrBlSoARw/viewform) has a script to automatically create a pull request for the new businesses. This script is:

```javascript
var ghToken = "PERSONAL_ACCESS_TOKEN";
var ghURI = "https://api.github.com/repos/kratsg/deaf-owned-businesses";

function sanitize_string(input){
  return input.replace(/[\W_]+/g,"").toLowerCase();
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

function get_business_file(ref){
  var options = {
    "method": "GET",
    "contentType": "application/json",
    "headers": {"Accept": "application/vnd.github.v3+json"}
  };

  var response = UrlFetchApp.fetch(ghURI + "/contents/_data/businesses.yml?access_token="+ghToken+"&ref="+ref, options);
  var json = response.getContentText();
  return JSON.parse(json);
}

function update_business_file(content, sha, branch_name, bizname, name, email){
  var payload = {
    "message": "add new business " + bizname,
    "committer": {
      "name": name,
      "email": email
    },
    "branch": branch_name,
    "content": content,
    "sha": sha
  };
  var options = {
    "method": "PUT",
    "contentType": "application/json",
    "payload": JSON.stringify(payload),
    "headers": {"Accept": "application/vnd.github.v3+json"}
  };

  var response = UrlFetchApp.fetch(ghURI + "/contents/_data/businesses.yml?access_token="+ghToken, options);
}

function create_pull_request(branch_name, bizname, body){
  var payload = {
    "title": "Add "+bizname,
    "body": "Please add " + bizname + " to the list of businesses. Thanks.\n\n" + body,
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

function add_new_business(bizname, new_business, body, timestamp, name, email){
  var branch_name = "add/business/"+sanitize_string(name)+"/"+sanitize_string(timestamp);
  create_branch(branch_name);

  var business_file = get_business_file(branch_name);
  Logger.log(business_file['content']);
  Logger.log(Utilities.base64Decode(business_file['content']));
  var content = Utilities.base64Encode(Utilities.newBlob(Utilities.base64Decode(business_file['content'], Utilities.Charset.UTF_8)).getDataAsString()+"\n"+new_business, Utilities.Charset.UTF_8);
  update_business_file(content, business_file['sha'], branch_name, bizname, name, email);
  create_pull_request(branch_name, bizname, body);
}

function create_issue(bizname, body){
  var payload = {
    "title": "Add "+bizname,
    "body": body,
    "labels": ["google forms", "enhancement", "good first issue", "help wanted"]
  };

  var options = {
    "method": "POST",
    "contentType": "application/json",
    "payload": JSON.stringify(payload),
    "headers": {"Accept": "application/vnd.github.v3+json"}
  };
  var response = UrlFetchApp.fetch(ghURI + "/issues?access_token="+ghToken, options);
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

  var new_business = "- category: " + category + "\n" +
                     "  comments: " + comments + "\n" +
                     "  location: " + location + "\n" +
                     "  name: " + bizname + "\n" +
                     "  url: " + url + "\n";

  var body = "```\n" + new_business +
             "```\n\n" +
             "Submitted by [" + name + "](mailto:" + email + ") at " + timestamp + " from [DeafBiz Google Form](https://docs.google.com/forms/d/e/1FAIpQLSdFPZyanfu-_nkWpKKwIIk15NzeoBXgkg5xSp4c5CrBlSoARw/viewform) automatically.";

  add_new_business(bizname, new_business, body, timestamp, name, email);

}
```
