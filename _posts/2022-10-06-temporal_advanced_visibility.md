--- 
layout: single
title:  "Temporal Advanced Visibility"
categories:
- Temporal
tags:
- Advanced Visibility
- Temporal
- Temporal Cloud
- Opensource
- Go
---

![Temporal](/assets/2022-08-15/logo-temporal-with-copy.svg)
## Overview
In this article we will show how to use Temporal advanced visibility. In Temporal you can have many, many workflows, billions in fact. Being able to query and identify workflows based on custom search attributes is what advanced visibility is all about. Custom search attributes can be of a type keyword, bool, int, datetime, double or text. Once a custom search attribute is defined, it can be set to a value, within your workflow code. Temporal will then store these attributes and their corresponding values in elasticsearch. Temporal cloud provides elasticsearch already integrated so no need to self-host your own elasticsearch. Simply create the custom search attributes and set them in your workflow.

## Use Case
In this example I have created a [backup application](https://github.com/ktenzer/temporal-demo-apps/tree/main/backup). Since a backup can fail for a lot of reasons, that is considered a normal outcome of a workflow. In general we should never fail workflows because our business logic failed. Without advanced visibility however, it would be difficult to get a list of failed backup workflows. I want to easily report failed or succeeded backups to the end user.

## Create Custom Search Attribute
Using the Temporal Cloud I create a custom search attribute under the namespace settings called BackupStatus. Since the value is a string I would set it to type Keyword.
![Custom Search Attribute](/assets/2022-10-06/csa.png)

## Set Custom Search Attribute
Now that I have created a custom search attribute I can simply set it in my workflow. In this case I want to set it to 'failed' in the event of a backup failure or 'succeeded' in the event of a backup success.
The Temporal SDK provides an API called 'UpsertSearchAttributes'. In the example below, I am setting 'BackupStatus' custom search attribute to 'failed'.

```go
attributes := map[string]interface{}{
	"BackupStatus": "failed",
}
workflow.UpsertSearchAttributes(ctx, attributes)
err := workflow.UpsertSearchAttributes(ctx, attributes)
if err != nil {
	return err
}
```

## Query Using Custom Search Attributes
After workflow upserts custom search attributes, we can query workflows using those attributes. In this example we will do an advanced query for 'failed' backup workflows using the Temporal cloud.
![Advanced Query](/assets/2022-10-06/query.png)

## Summary
In this article we learned about the value of using custom search attributes in Temporal. Allowing developers to easily identify and query workflows based on custom search attributes. The Temporal cloud abstracts away the complexity of hosting elasticsearch, making the feature seamless and easy to consume. This is a key capability allowing developers to not fail workflows because business logic failed.

(c) 2022 Keith Tenzer




