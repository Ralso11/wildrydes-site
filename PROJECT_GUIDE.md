# Project Guide: Wild Rydes on AWS

A step-by-step walkthrough of how I built the app in the AWS Console, in `eu-central-1` (Frankfurt). Each step says what to do, where, and why. The last sections list the problems I hit.

**Time needed:** about 2 hours. **Cost:** inside the Free Tier for a short test, as long as you clean up at the end.

---

## 0. Prerequisites

- An AWS account with a budget alert, and a user with admin access
- A GitHub account
- A text file to keep notes: you will copy several IDs and URLs
- One region for everything. I used **Europe (Frankfurt) `eu-central-1`**. Check the region in the top-right corner on every service.

> **Naming:** I added `2024` to some names (for example the table `Rides2024`) because I had older versions. Use any names you like, but the table name in the Lambda code must match your table exactly.

## 1. GitHub: your own copy of the site

1. If you already have an old, unfinished copy of the project, rename it (for example `wild-rides-old`) so the name is free.
2. Open the template repo `tinytechnicaltutorials/wildrydes-site`.
3. Click **Use this template → Create a new repository**, name it `wildrydes-site`, make it **Public**.

## 2. Amplify: host the site and deploy on every commit

1. **AWS Amplify → Create new app**, choose **GitHub** as the source.
2. Authorize Amplify and select the `wildrydes-site` repo and the `main` branch. (If the repo doesn't show up, update the GitHub permissions and refresh. It may take a couple of tries.)
3. Accept the defaults, **Save and deploy**.
4. Open the domain link. You should see the Wild Rydes landing page.

From now on, every commit to `main` redeploys the site automatically.

## 3. Cognito: registration and login

**Why:** users need to register and log in. Cognito stores them and issues tokens.

> ⚠️ **User pools, not identity pools.** The Cognito start page can send you to *Create identity pool*. That is a different service. You need **User pools**.

1. Open **Amazon Cognito → User pools → Create user pool** (use **View all** in the left menu if you land on a single pool).
2. Application type: **Single-page application (SPA)**. Application name: `wild-rydes-web-app`.
3. Sign-in identifier: **Username** only (no email or phone as sign-in option).
4. Required attributes: **email** (the wizard demands one when you use a username). Leave **self-registration** on.
5. Create it. Rename the pool to `WildRydes` (optional).
6. Open **App clients → wild-rydes-web-app** and check:
   - **Client ID** (copy it).
   - **Authentication flows:** `ALLOW_USER_SRP_AUTH` must be ticked. The site logs in with SRP from the browser.
7. On the user pool **Overview**, copy the **User pool ID**.
8. Check **Sign-in → MFA = No MFA** and **email delivery = Send email with Cognito**.

### Put the IDs into the site

In your GitHub repo, edit `js/config.js`:

```js
window._config = {
    cognito: {
        userPoolId: 'eu-central-1_XXXXXXXXX',
        userPoolClientId: 'YOUR_APP_CLIENT_ID',
        region: 'eu-central-1'
    },
    api: {
        invokeUrl: ''   // filled in later, after API Gateway
    }
};
```

Commit. Amplify redeploys. The region must be the region of your **user pool**.

**Test:** open `<your-amplify-domain>/register.html`, create an account (use a real email, Cognito sends a code), verify it, then log in at `/signin.html`. After login the page shows a long **ID token**. You can use it later to test the API authorizer.

## 4. DynamoDB: the rides table

1. **DynamoDB → Create table**.
2. Table name: `Rides2024` (any name, but remember it).
3. Partition key: **`RideId`**, type **String** (exact spelling, the Lambda code uses it).
4. Defaults for everything else. Create.
5. Open the table → **Additional info** and copy the **ARN**: `arn:aws:dynamodb:eu-central-1:<account-id>:table/Rides2024`. No trailing space.

## 5. IAM: a role the Lambda can use

**Why:** by default a Lambda can't write to DynamoDB. The role grants exactly that.

1. **IAM → Roles → Create role**. Trusted entity: **AWS service**, use case **Lambda**.
2. Attach the managed policy **AWSLambdaBasicExecutionRole** (it lets the function write logs). Name the role `WildRydesLambda`.
3. Open the new role → **Add permissions → Create inline policy** → **JSON**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "dynamodb:PutItem",
      "Resource": "arn:aws:dynamodb:eu-central-1:<account-id>:table/Rides2024"
    }
  ]
}
```

4. Name it `DynamoDBWriteAccess` and create it. Only `PutItem`, on this one table (least privilege).

## 6. Lambda: the ride logic

1. **Lambda → Create function → Author from scratch**.
2. Name: `RequestUnicorn`. Runtime: latest **Node.js**.
3. Under *Change default execution role* choose **Use an existing role** and select `WildRydesLambda`.
4. Replace the code with the function from `README-template.md` (the template's Lambda code).
5. **Change the table name** in the code to your table (for example `Rides2024`). If this doesn't match, nothing works.
6. Click **Deploy**.
7. Create a test event with the test JSON from `README-template.md`, save it and click **Test**. You should get status **201** with a unicorn in the response, and a new item in DynamoDB (**Explore table items**).

## 7. API Gateway: the public endpoint

1. **API Gateway → Create API → REST API → Build**, name `WildRydes`.
2. **Authorizers → Create authorizer:** name `WildRydes`, type **Cognito**, choose your user pool (check the region), token source `Authorization`.
3. In the authorizer, use **Test authorizer** with the **ID token** from your sign-in page. A fresh token returns `200`.
4. **Resources → Create resource:** path `/`, name `ride`, tick **CORS**.
5. With `/ride` selected: **Create method → POST**, integration type **Lambda function**, tick **Lambda proxy integration**, pick `RequestUnicorn`.
6. Open the method's **Method request → Edit** and set **Authorization** to the Cognito authorizer `WildRydes`. Save.
7. **Deploy API:** new stage `dev`. Copy the **Invoke URL**, for example `https://abc123.execute-api.eu-central-1.amazonaws.com/dev`.

### Connect the site to the API

In `js/config.js` set the URL (no trailing slash, and without `/ride`):

```js
api: {
    invokeUrl: 'https://abc123.execute-api.eu-central-1.amazonaws.com/dev'
}
```

Commit and wait for the Amplify deployment.

## 8. Test end to end

1. Open `<your-amplify-domain>/signin.html` and log in.
2. Open `/ride.html`, click on the map, press **Set Pickup**.
3. You should see "Your unicorn is on the way" and then "has arrived. Giddy up!".
4. Check **DynamoDB → Explore table items → Run**: there is one item per ride, with `RideId`, `RequestTime`, `Unicorn` and `User`.

---

## Problems I hit (and the fixes)

| Symptom | Cause | Fix |
|---|---|---|
| Cognito shows **Create identity pool** and no user pool | I was in the *identity pools* part of Cognito | Go to **User pools** (left menu → View all → Create user pool) |
| Wizard error: *"Username sign-in requires that you choose email or phone as a required attribute"* | Required attributes were empty | Select **email** as required attribute |
| I couldn't find **App integration** in the user pool | New console moved it | **Applications → App clients** has the Client ID |
| Login fails with an auth-flow error | `ALLOW_USER_SRP_AUTH` not enabled | Enable it on the app client |
| Authorizer **Test** returns `401 Unauthorized` | The ID token had expired (valid for 60 minutes) or was copied with a changed character | Sign in again and paste a **fresh** token straight from the page, with no spaces or `Bearer ` prefix |
| Only the register page opens, I can't sign in | The landing page only links to register | Open `/signin.html` directly |
| Lambda returns an error about the table | Table name in the code ≠ real table name | Make code, policy and DynamoDB use exactly the same name (`Rides2024`) and key `RideId` |
| `AccessDeniedException` from DynamoDB | IAM policy missing, or wrong table ARN (watch for a trailing space) | Fix the `Resource` ARN of the inline policy |
| Browser error about CORS | CORS not enabled, or API not redeployed | Enable CORS on `/ride` and **Deploy API** again |
| Changes in `config.js` have no effect | Amplify hadn't finished the new deployment, or the browser cache | Wait for the deployment to turn green, then hard-refresh (Ctrl+F5) |

---

## Clean-up (avoid surprise bills)

Delete in this order:

1. **Amplify:** the app (App settings → General settings → Delete app)
2. **API Gateway:** the API
3. **Cognito:** the user pool
4. **Lambda:** the function
5. **IAM:** the role (and its policies)
6. **DynamoDB:** the table (no backup, delete its alarms)
7. **CloudWatch → Log groups:** `/aws/lambda/RequestUnicorn`
8. **GitHub:** the repo stays (it costs nothing)

## Improvements

- Rebuild with **Terraform** and a CI/CD pipeline instead of clicking in the console.
- Restrict **CORS** from `*` to the site's own domain.
- Add **throttling** or a usage plan to the API.
- Make the unicorn choice smarter (nearest unicorn) and show ride history from DynamoDB.
- The sample site keeps the config in `config.js`. In a real app, deploy configuration through environment variables.

## Credits

Wild Rydes is an AWS sample application. The tutorial that guided this build is by **Tiny Technical Tutorials** (YouTube). This guide and the troubleshooting notes are my own write-up.
