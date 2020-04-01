## Task 1 - A Schema Design to Represent The Subscription Based Service

### Introduction

Although the brief asked for a single table to store the data. This seemed wrong for a number of reasons, especially if we're going to store our data in a relational database.

I felt that it would be better to break entities up for extensibility and the querying of data.

I've chosen to define my data structure in terms of dotnet POCOs (Plain old CLR objects). I'm also making some assumptions that an ORM (I'm familiar with Entity Framework) is used to map the classes to our database schema. I've not included all of the attributes as I felt it more important to just demonstrate the structure.

> 💬 Note: I've not "gone to town" by laying out every little part but I feel that I've put enough below to demonstrate my approach.

### Subscription Model

Let's consider a Subscription, each subscription could look like so:

```csharp

public class Subscription
{
    /// <summary>
    /// A Unique ID for the subscription
    /// </summary>
    public int Id { get; set; }

    /// <summary>
    ///  The subscriber either an individual or perhaps a business.
    /// </summary>
    public Subscriber Subscriber { get; set; }

    /// <summary>
    /// The subscription tier typically free, paid, etc.
    /// </summary>
    public SubscriptionTier SubscriptionTier { get; set; }

    /// <summary>
    /// The subscription type typically monthly, yearly, etc.
    /// </summary>
    /// <remarks>
    ///  Synonymous with billing cycle.
    /// </remarks>
    public SubscriptionType SubscriptionType { get; set; }

    /// <summary>
    /// The subscription creation date.
    /// </summary>
    public DateTime Commencement { get; set; }

    /// <summary>
    /// The latest invoice created date
    /// </summary>
    /// <remarks>
    /// Used in conjunction with SubscriptionType to calculate
    /// the next billing date.
    /// </remarks>
    public DateTime LastInvoiceDate { get; set; }

    /// <summary>
    /// Stores the billing details for the subscription.
    /// </summary>
    public BillingDetails BillingDetails { get; set; }

    /// <summary>
    /// Used for third party checking of subscriptions.
    /// </summary>
    public Boolean IsValid { get; set; }

}
```

- A Subscriber doesn't necessarily have to be an individual user. By abstracting
  this away we could support Business to Business transactions with little to zero
  to the schema.

- The above would enable a Subscriber to have multiple subscriptions each with their own billing details. I think that this would be advantageous in the longer term especially if the business expand their service offerings.

### Billing Details Model

- Billing details could in turn have their own payment methods which are extendable. These would represent the various payment processors.

- Account reference could be an encrypted token used to identify the user on the third party platform.

```csharp

public class BillingDetails
{
    public int Id { get; set; }
    public PaymentMethod PaymentMethod { get; set; }
    public string AccountRef { get; set;}
}

```

## Task 2 - API Endpoint To Get Subscription Information

GET https://www.yourdomain.com/subscriptions/{user}

* The GET http method would be used to call the api.

* Subscriptions by user was chosen as this would create a nice neat restful entity (Subscriptions) for which we can make a Subscriptions Controller.

## Task 3 - Managing Subscriptions

<img src="./assets/images/authflow.svg"></img>

#### Assumptions / Concerns
My biggest convern is what happens if the user cancels their subscription after logging in? In theory a malicious user could make changes that would temporarily grant them access to a higher teirs worth of content. They could do this a few times though. At the moment we only have a two teir subscription model so it's not a problem but this could present a problem in the future so we should be thinking about this now.

💡 We should additionally check the subscriptions when teir information is changed AND when content is accessed.

If the user subscription is not valid we would update the IsValid field on the subscription table (as shown in task 1)

## Task 4 - Write a Function to Filter Customers

**Requirements:**
Customers:
- Are subscribed monthly
- AND have a valid subscription
- AND the subscriptions were created the previous calendar month
- The customers should be ordered by the earliest subscription start date

> 💬 I've chosen to write this in SQL as I think it's perfectly designed for this task. We could also use Linq expressions along with an IQueryable interface for deffered execution but I thought this would be much easier to express in SQL for the purposes of answering this question.
``` sql
SELECT Subscribers.ID
FROM Subscribers
INNER JOIN Subscriptions ON Users.ID=Subscriptions.ID
AND Subscriptions.IsValid=1
AND DATEPART(m, Subscriptions.Commencement) = DATEPART(m, DATEADD(m, -1, getdate()))

order by convert(datetime, Subscriptions.Commencement, 103) ASC
```

#### Assumptions
- We only require the Subscriber IDs.
- We're using the schema I specified above where subscribers are customers.
