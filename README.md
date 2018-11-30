# Submitting a business

To submit a new business, edit [_data/businesses.yml](_data/businesses.yml) to add the new entry. Then submit a [pull request](https://github.com/kratsg/pulls) to get it on the website.

Feel free to file an issue instead of a pull request.

# Google Forms Automation

The [google form](https://docs.google.com/forms/d/e/1FAIpQLSdFPZyanfu-_nkWpKKwIIk15NzeoBXgkg5xSp4c5CrBlSoARw/viewform) has a script to automatically file an issue on this respository for new businesses. This script is (partially redacted):

```
var ghToken = "PERSONAL_ACCESS_TOKEN";

function onFormSubmit(e) {

  var title = "New Business Request";
  var labels = ["google forms", "enhancement", "good first issue", "help wanted"];

  var timestamp = e.values[0];
  var email = e.values[1];
  var name = e.values[2];

  var bizname = e.values[3];
  var url = e.values[4];
  var location = (e.values[5] == "")?"null":e.values[4];
  var comments = (e.values[6] == "")?"null":e.values[5];
  var category = (e.values[7] == "")?"null":e.values[6];

  var body = "```\n" +
             "- category: " + category + "\n" +
             "  comments: " + comments + "\n" +
             "  location: " + location + "\n" +
             "  name: " + bizname + "\n" +
             "  url: " + url + "\n" +
             "```\n\n" +
             "Submitted by [" + name + "](mailto:" + email + ") at " + timestamp + " from [DeafBiz Google Form](https://docs.google.com/forms/d/e/1FAIpQLSdFPZyanfu-_nkWpKKwIIk15NzeoBXgkg5xSp4c5CrBlSoARw/viewform) automatically.";

  var payload = {
    "title": title,
    "body": body,
    "labels": labels
  };

  var options = {
    "method": "POST",
    "contentType": "application/json",
    "payload": JSON.stringify(payload),
    "muteHttpExceptions": true,
    "headers": {"Accept": "application/vnd.github.v3+json"}
  };
  var response = UrlFetchApp.fetch("https://api.github.com/repos/kratsg/deaf-owned-businesses/issues?access_token="+ghToken, options);

  console.log({'payload': payload, 'options': options, 'response': response});
}
```
