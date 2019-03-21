
# Build a .NET CLI app with Microsoft Graph

In this lab you will create a .NET application, configured with Azure Active Directory (Azure AD) for authentication and authorization using Microsoft Authentication Library (MSAL).

## In this lab
- [Exercise 1: Create a CLI application](#exercise-1-create-a-cli-application)
- [Exercise 2: Register a native application with the Azure portal](#exercise-2-register-a-native-application-with-the-azure-portal)
- [Exercise 3: Extend the app for Azure AD Authentication](#exercise-3-extend-the-app-for-azure-ad-authentication)
- [Exercise 4: Extend the app for Microsoft Graph](#exercise-4-extend-the-app-for-microsoft-graph)
- [Exercise 5: Schedule an event with Graph SDK](#exercise-5-schedule-an-event-with-graph-sdk)
- [Exercise 6: Book a room for an event](#exercise-6-book-a-room-for-an-event)
- [Exercise 7: Schedule a recurrent event](#exercise-7-schedule-a-recurrent-event)
- [Exercise 8: Schedule an all day event](#exercise-8-schedule-an-all-day-event)
- [Exercise 9: Accept an invite to an event](#exercise-9-accept-an-invite-to-an-event) 
- [Exercise 10: Decline an invite to an event](#exercise-10-decline-an-invite-to-an-event)

## Prerequisites

To complete this lab, you need the following:

- [Visual Studio](https://visualstudio.microsoft.com/vs/) installed on your development machine. (**Note**: This tutorial was written with Visual Studio 2017 version 15.8.3.)

If you don't have a Microsoft account, there are a couple of options to get a free account:
- You can [sign up for a new personal Microsoft account](https://signup.live.com/signup?wa=wsignin1.0&rpsnv=12&ct=1454618383&rver=6.4.6456.0&wp=MBI_SSL_SHARED&wreply=https://mail.live.com/default.aspx&id=64855&cbcxt=mai&bk=1454618383&uiflavor=web&uaid=b213a65b4fdc484382b6622b3ecaa547&mkt=E-US&lc=1033&lic=1)
- You can [sign up for the Office 365 Developer Program](https://developer.microsoft.com/office/dev-program) to get  a free Office 365 subscription

## Exercise 1: Create a CLI application

Open Visual Studio, and select **File > New > Project.** In the **New Project** dialog, do the following:
1. Select **Visual C# > Get Started**.
2. Select **Console App**.
3. Enter **Calendar** for the name of the project.

> ### Important
> Ensure that you enter the exact name for the Visual Studio Project that is specified in these lab instructions. The project name will also be the **namespace**. If you used a different project name, adjust all the namespaces to match your project name.

Press **Ctrl + F5** to run the application or **Debug > Start Debugging** to run the application in debug mode. If everything is working fine a terminal window will open.

Before moving on, install additional NuGet packages that you will use later.

Select **Tools > Nuget Package Manager > Package Manager Console**. In the console window, run the following commands:
```powershell
Install-Package "Microsoft Graph"
Install-Package "Microsoft.Identity.Client"
Install-Package "System.Configuration.ConfigurationManager"
```

## Exercise 2: Register a native application with the Azure portal
In this exercise, you will create a new Azure AD native application using the Application Registry Portal.

1. Open a browser and navigate to the [Azure portal app registrations page](). Login using a **personal account**(aka: Microsoft Account) or **Work or School Account**.
2. Select **New registration** at the top of the page.
3. On the Register an application page, set the **Application Name** to **Calendar**. Under "Supported account types", choose **Accounts in any organizational directory and personal Microsoft accounts**.
4. Select **Register** at the bottom of the page.
5. On the **Calendar** page, copy the **Application (client) ID** as you will need it later.
6. Go to the **Authentication** page. Under **Suggested Redirect URIs for public clients (mobile, desktop)** check the boxes next to **urn:ietf:wg:oauth:2.0:oob**  and **https://login.microsoftonline.com/common/oauth2/nativeclient**.
7. Choose **Save** near the top of the Authentication page.

## Exercise 3: Extend the app for Azure AD Authentication
In this exercise you will extend the application from **Exercise 1** to support authentication with Azure AD. This is required to obtain the necessary OAuth token
to call the Microsoft Graph.

Edit the **app.config** file, and immediately before the `/configuration` element, add the following element:
```xml
<appSettings>
    <add key="clientId" value="THE_APPLICATION_ID_YOU_COPIED_IN_EXERCISE_2">
</appSettings>
```

## Add GraphClientServiceProvider.cs
1. Add a class to the project named **GraphClientServiceProvider** This class will be responsible for authenticating using the Microsoft Authentication Library (MSAL), which is the
Microsoft.Identity.Client package that we installed. For separation of concerns, change the **namespace** of this class to **Helpers**

2. Replace the `using` statement at the top of the file
```csharp
using Microsoft.Graph;
using Microsoft.Identity.Client;
using System;
using System.Collections.Generic;
using System.Configuration;
using System.Diagnostics;
using System.Net.Http.Headers;
using System.Threading.Tasks;
```

3. Replace the `class` declaration with the following
```csharp
class GraphClientServiceProvider
    {
        // The client ID is used by the application to uniquely identify itself to the authentication endpoint.
        private static readonly string clientId = ConfigurationManager.AppSettings["clientId"].ToString();
        private static readonly string[] scopes = {
            "https://graph.microsoft.com/User.Read"
        };

        private static PublicClientApplication identityClientApp = new PublicClientApplication(clientId);
        private static GraphServiceClient graphClient = null;

        // Get an access token for the given context and resourceId. An attempt is first made to acquire the token silently.
        // If that fails, then we try to acquire the token by prompting the user.
        public static GraphServiceClient GetAuthenticatedClient()
        {
            if (graphClient == null)
            {
                try
                {
                    graphClient = new GraphServiceClient(
                        "https://graph.microsoft.com/v1.0",
                        new DelegateAuthenticationProvider(
                                async (requestMessage) =>
                                {
                                    var token = await GetTokenForUserAsync();
                                    requestMessage.Headers.Authorization = new AuthenticationHeaderValue("bearer", token);
                                }
                            ));
                    return graphClient;
                } 
                catch(Exception error)
                {
                    Debug.WriteLine($"Could not create a graph client {error.Message}");
                }
            }
            return graphClient;
        }

        /// <summary>
        /// Get token for User
        /// </summary>
        /// <returns>Token for User</returns>
        private static async Task<string> GetTokenForUserAsync()
        {
            AuthenticationResult authResult = null;

            try
            {
                IEnumerable<IAccount> account = await identityClientApp.GetAccountsAsync();
                authResult = await identityClientApp.AcquireTokenSilentAsync(scopes, account as IAccount);
                return authResult.AccessToken;
            }
            catch(MsalUiRequiredException)
            {
                // This means the AcquireTokenSilentAsync threw an exception. 
                // This prompts the user to log in with their account so that we can get the token.
                authResult = await identityClientApp.AcquireTokenAsync(scopes);
                return authResult.AccessToken;
            }
        }

    }
```

## Exercise 4: Extend the app for Microsoft Graph
In this exercise you will incorporate the Microsoft Graph into the application. For this application, you will use the [Microsoft Graph Client Library for .NET](https://github.com/microsoftgraph/msgraph-sdk-dotnet) to make calls to Microsoft Graph.
Add the following code snippet below **Main** in `Program.cs`

```csharp
        /// <summary>
        /// Gets a User from Microsoft Graph
        /// </summary>
        /// <returns>A User object</returns>
        public static async Task<User> GetMeAsync()
        {
            User currentUser = null;
            try
            {
                var graphClient = Authentication.GetAuthenticatedClient();

                // Request to get the current logged in user object from Microsoft Graph
                currentUser = await graphClient.Me.Request().GetAsync();

                return currentUser;
            }

            catch (ServiceException e)
            {
                Debug.WriteLine("We could not get the current user: " + e.Error.Message);
                return null;
            }
        }

        static async Task RunAsync()
        {
            var me = await GetMeAsync();

            Console.WriteLine($"{me.DisplayName} logged in.");
            Console.WriteLine();
        }
```

Replace the **Main** function with the following
```csharp
        static void Main(string[] args)
        {
            Console.WriteLine("Welcome to the Calendar CLI");

            RunAsync().GetAwaiter().GetResult();
            Console.ReadKey();
        }
```

Press **Ctrl + F5** to run the application or **Debug > Start Debugging** to run the application in debug mode. If everything is working fine a terminal window will show and prompt you to log in.

## Exercise 5: Schedule an event with Graph SDK
1. In this exercise you will schedule a meeting using the **Microsoft Graph client libray for .NET**. Create a class called **CalendarController**, this class is going to abstract the Graph SDK  functionality.

2. Before you go on, give your application permissions to interact with the Calendar. Go to the [developer portal](https://apps.dev.microsoft.com/#/application/fb43c824-ceab-43f8-b079-e70d8224c0a1)
   scroll down to **Microsoft Graph Permissions**. Click on **Add** at **Delegated Permissions** and select **Calendars.ReadWrite**

3. In the `GraphClientServiceProvider` class you created in the previous exercise replace the **scope** variable declaration with the following
```csharp
        private static string[] scopes = {
            "https://graph.microsoft.com/User.Read",
            "https://graph.microsoft.com/Calendars.ReadWrite"
        };
```

2. Replace the `using` statement at the top of the file
```csharp
using Microsoft.Graph;
using System;
using System.Threading.Tasks;
```

3. Replace the `class` declaration with the following
```csharp
class CalendarController
    {
        GraphServiceClient graphClient;

        public CalendarController(GraphServiceClient client)
        {
            graphClient = client;
        }

        /// <summary>
        /// Schedules an event.
        /// 
        /// For purposes of simplicity we only allow the user to enter the startTime
        /// and endTime as hours.
        /// </summary>
        /// <param name="subject">Subject of the meeting</param>
        /// <param name="startTime">The time when the meeting starts</param>
        /// <param name="endTime">Duration of the meeting</param>
        /// <param name="attendeeEmail">Email of person to invite</param>
        /// <returns></returns>
        public async Task ScheduleEventAsync(string subject, string startTime, string endTime, string attendeeEmail)
        {
            DateTime dateTime = DateTime.Today;

            // set the start and end time for the event
            DateTimeTimeZone start = new DateTimeTimeZone
            {
                TimeZone = "Pacific Standard Time",
                DateTime = $"{dateTime.Year}-{dateTime.Month}-{dateTime.Day}T{startTime}:00:00"
            };
            DateTimeTimeZone end = new DateTimeTimeZone
            {
                TimeZone = "Pacific Standard Time",
                DateTime = $"{dateTime.Year}-{dateTime.Month}-{dateTime.Day}T{endTime}:00:00"
            };

            // Adds attendee to the event
            EmailAddress email = new EmailAddress
            {
                Address = attendeeEmail
            };

            Attendee attendee = new Attendee
            {
                EmailAddress = email,
                Type = AttendeeType.Required,
            };
            List<Attendee> attendees = new List<Attendee>();
            attendees.Add(attendee);

            Event newEvent = new Event
            {
                Subject = subject,
                Attendees = attendees,
                Start = start,
                End = end
            };

            try
            {
                /**
                 * This is the same as a post request 
                 * 
                 * POST: https://graph.microsoft.com/v1.0/me/events
                 * Request Body
                 * {
                 *      "subject": <event-subject>
                 *      "start": {
                            "dateTime": "<date-string>",
                            "timeZone": "Pacific Standard Time"
                          },
                         "end": {
                             "dateTime": "<date-string>",
                             "timeZone": "Pacific Standard Time"
                          },
                          "attendees": [{
                            emailAddress: {
                                address: attendeeEmail 
                            }
                            "type": "required"
                          }]
                            
                 * }
                 * 
                 * Learn more about the properties of an Event object in the link below
                 * https://developer.microsoft.com/en-us/graph/docs/api-reference/v1.0/resources/event
                 * */
                Event calendarEvent = await graphClient
                    .Me
                    .Events
                    .Request()
                    .AddAsync(newEvent);

                Console.WriteLine($"Added {calendarEvent.Subject}");
            }
            catch (ServiceException error)
            {
                Console.WriteLine(error.Message);
            }

        }
    }
```

#### Putting things together
1. Replace the content of the **Main** function with the following
```csharp
            graphClient = GraphServiceClientProvider.GetAuthenticatedClient();
            cal = new CalendarController(graphClient);
            RunAsync().GetAwaiter().GetResult();

            Console.WriteLine("Available commands:\n" +
                "\t 1. schedule-event \n " +
                "\t exit");
            var command = "";

            do
            {
                Console.Write("> ");
                command = Console.ReadLine();
                if (command != "exit") runAsync(command).GetAwaiter().GetResult();
            }
            while (command != "exit");
```
This allows the user interact with your application through the command line interface.

2. Add this function below **Main**
```csharp
       private static async Task runAsync(string command)
        {
            switch (command)
            {
                case "schedule-event":
                    Console.WriteLine("Enter the subject of your event");
                    var subject = Console.ReadLine();

                    Console.WriteLine("Invite an attendee to this event, enter their email");
                    var attendee = Console.ReadLine();

                    Console.WriteLine("Enter the start time of your event, in 24hr format 00 - 23");
                    var startTime = Console.ReadLine().Substring(0, 2);

                    Console.WriteLine("Enter the end time of your event, in 24hr format 00 - 23");
                    var endTime = Console.ReadLine().Substring(0, 2);

                    await cal.ScheduleMeetingAsync(subject, startTime, endTime, attendee);
                    break;
                default:
                    Console.WriteLine("Invalid command");
                    break;
            }
        }
```
This function uses the CalendarController to interact with the Graph SDK.

## Exercise 6: Book a room for an event
In this exercise you are going to book a room for an event.
1. In `CalendarController.cs` add the function below
```csharp
        /// <summary>
        /// Books a room for the event
        /// </summary>
        /// <param name="eventId"></param>
        /// <param name="resourceMail"></param>
        /// <returns></returns>
        public async Task BookRoomAsync(string eventId, string resourceMail)
        {
            /**
             * A room is an an attendee of type resource
             * 
             * Refer to the link below to learn more about the properties of the Attendee class
             * https://developer.microsoft.com/en-us/graph/docs/api-reference/v1.0/resources/attendee
             **/
            Attendee room = new Attendee();
            EmailAddress email = new EmailAddress();
            email.Address = resourceMail;
            room.Type = AttendeeType.Resource;
            room.EmailAddress = email;

            List<Attendee> attendees = new List<Attendee>();
            Event patchEvent = new Event();

            attendees.Add(room);
            patchEvent.Attendees = attendees;

            try
            {
                /**
                 * This is the same as making a patch request
                 * 
                 * PATCH https://graph.microsoft.com/v1.0/me/events/{id}
                 * 
                 * request body 
                 * {
                 *      attendees: [{
                 *              emailAddress: {
                 *                  "address": "email@address.com"
                 *              },
                 *              type: "resource"
                 *          }
                 *      ]
                 * }
                 * */
                 await graphClient
                    .Me
                    .Events[eventId]
                    .Request()
                    .UpdateAsync(patchEvent);
            }
            catch (Exception error)
            {
                Console.WriteLine(error.Message);
            }
        }
```
2. Add the **book-room** command to the list of available commands in the **main** function
```csharp
 "\t 2. book-room\n " + 
```

3. Add the following **case** statement to the **runAsync** function in **Program.cs**
```csharp
case "book-room":
    Console.WriteLine("Enter the event id");
    var eventId = Console.ReadLine();

    Console.WriteLine("Enter the resource email");
    var resourceEmail = Console.ReadLine();

    await cal.BookRoomAsync(eventId, resourceEmail);
    break;
```

This prompts the user to enter the **eventId** and **resource email**.

## Exercise 7: Schedule a recurrent event
In this exercise you are going to set a recurring event.

1. Add the code below to **CalendarController.cs**
```csharp
        /// <summary>
        /// Sets recurrent events
        /// </summary>
        /// <param name="subject"></param>
        /// <param name="startDate"></param>
        /// <param name="endDate"></param>
        /// <param name="startTime"></param>
        /// <param name="endTime"></param>
        /// <returns></returns>
        public async Task SetRecurrentAsync(string subject, string startDate, string endDate, string startTime, string endTime)
        {
            // Sets the event to happen every week
            RecurrencePattern pattern = new RecurrencePattern
            {
                Type = RecurrencePatternType.Weekly,
                Interval = 1
            };

            /**
             * Sets the days of the week the event occurs.
             * 
             * For this sample it occurs every Monday
             ***/
            List<Microsoft.Graph.DayOfWeek> daysOfWeek = new List<Microsoft.Graph.DayOfWeek>();
            daysOfWeek.Add(Microsoft.Graph.DayOfWeek.Monday);
            pattern.DaysOfWeek = daysOfWeek;

            /**
             * Sets the duration of time the event will keep recurring.
             * 
             * In this case the event runs from Nov 6th to Nov 26th 2018.
             **/
            int startDay = int.Parse(startDate.Substring(0, 2));
            int startMonth = int.Parse(startDate.Substring(3, 2));
            int startYear = int.Parse(startDate.Substring(6, 4));

            int endDay = int.Parse(endDate.Substring(0, 2));
            int endMonth = int.Parse(endDate.Substring(3, 2));
            int endYear = int.Parse(endDate.Substring(6, 4));

            RecurrenceRange range = new RecurrenceRange
            {
                Type = RecurrenceRangeType.EndDate,
                StartDate = new Date(startYear, startMonth, startDay),
                EndDate = new Date(endYear, endMonth, endDay)
            };

            /**
             * This brings together the recurrence pattern and the range to define the
             * PatternedRecurrence property.
             **/
            PatternedRecurrence recurrence = new PatternedRecurrence
            {
                Pattern = pattern,
                Range = range
            };

            DateTime dateTime = DateTime.Today;
            // set the start and end time for the event
            DateTimeTimeZone start = new DateTimeTimeZone
            {
                TimeZone = "Pacific Standard Time",
                DateTime = $"{startYear}-{startMonth}-{startDay}T{startTime}:00:00"
            };

            DateTimeTimeZone end = new DateTimeTimeZone
            {
                TimeZone = "Pacific Standard Time",
                DateTime = $"{startYear}-{startMonth}-{startDay}T{startTime}:00:00"
            };

            Event eventObj = new Event
            {
                Recurrence = recurrence,
                Subject = subject,
            };

            try
            {
                var recurrentEvent = await graphClient
                    .Me
                    .Events
                    .Request()
                    .AddAsync(eventObj);
                Console.WriteLine($"Created {recurrentEvent.Subject}," +
                    $" happens every week on Monday from {startTime}:00 to {endTime}:00");
            }
            catch (Exception error)
            {
                Console.WriteLine(error.Message);
            }
        }
```

2. Add the **schedule-recurrent-event** command to the list of available commands in the **main** function
```csharp
 "\t 3. recurrent-event \n " + 
```

3. Add the following **case** statement in the **runAsync** method
```csharp
    case "schedule-recurrent-event":
        Console.WriteLine("Enter the event subject");
        var eventSubject = Console.ReadLine();

        Console.WriteLine("Enter the start date of your event DD/MM/YYYY");
        var startDate = Console.ReadLine();

        Console.WriteLine("Enter the end date of your event DD/MM/YYYY");
        var endDate = Console.ReadLine();

        Console.WriteLine("Enter the start time of your event, in 24hr format 0000 - 2300");
        var startRecurrent = Console.ReadLine().Substring(0, 2);

        Console.WriteLine("Enter the end time of your event, in 24hr format 0000 - 2300");
        var endRecurrent = Console.ReadLine().Substring(0, 2);

        await cal.SetRecurrentAsync(eventSubject, startDate, endDate, startRecurrent, endRecurrent);
        break;
```

## Exercise 8: Schedule an all day event
In this exercise you're going to create an all day event.

1. Add the code below to **CalendarController.cs**
```csharp
        /// <summary>
        /// Sets all day events
        /// </summary>
        /// <param name="eventSubject"></param>
        /// <param name="attendeeEmail"></param>
        /// <param name="date"></param>
        /// <returns></returns>
        public async Task SetAllDayAsync(string eventSubject, string attendeeEmail, string date)
        {
            // Adds attendee to the event
            EmailAddress email = new EmailAddress
            {
                Address = attendeeEmail
            };

            Attendee attendee = new Attendee
            {
                EmailAddress = email,
                Type = AttendeeType.Required,
            };
            List<Attendee> attendees = new List<Attendee>();
            attendees.Add(attendee);

            int day = int.Parse(date.Substring(0, 2));
            int month = int.Parse(date.Substring(3, 2));
            int year = int.Parse(date.Substring(6, 4));

            Date allDayDate = new Date(year, month, day);
            DateTimeTimeZone start = new DateTimeTimeZone
            {
                TimeZone = "Pacific Standard Time",
                DateTime = allDayDate.ToString()
            };

            Date nextDay = new Date(year, month, day + 1);
            DateTimeTimeZone end = new DateTimeTimeZone
            {
                TimeZone = "Pacific Standard Time",
                DateTime = nextDay.ToString()
            };

            Event newEvent = new Event
            {
                Subject = eventSubject,
                Attendees = attendees,
                IsAllDay = true,
                Start = start,
                End = end
            };

            try
            {
                var allDayEvent = await graphClient
                    .Me
                    .Events
                    .Request()
                    .AddAsync(newEvent);

                Console.WriteLine($"Created an all day event: {newEvent.Subject}." +
                    $" Happening on {date}");
            }
            catch (Exception error)
            {
                Console.WriteLine(error.Message);
            }
        }
```

2. Add the **schedule-allday-event** command to the list of available commands in the **main** function
```csharp
 "\t 4. schedule-allday-event \n " + 
```

3. Add the following **case** statement in the **runAsync** method
```csharp
    case "schedule-allday-event":
        Console.WriteLine("Enter the event's subject");
        var allDaySubject = Console.ReadLine();

        Console.WriteLine("Invite an attendee to this event, enter their email");
        var allDayAttendee = Console.ReadLine();

        Console.WriteLine("Enter the date of your event DD/MM/YYYY");
        var allDayDate = Console.ReadLine();

        await cal.SetAllDayAsync(allDaySubject, allDayAttendee, allDayDate);
        break;
```

## Exercise 9: Accept an invite to an event
In this exercise you are going to accept an invite to an event.

1. Add the code below to **CalendarController.cs**
```csharp
        /// <summary>
        /// Accepts an event invite
        /// </summary>
        /// <param name="eventId"></param>
        /// <returns></returns>
        public async Task AcceptAsync(string eventId)
        {
            try
            {
                await graphClient
                      .Me
                      .Events[eventId]
                      .Accept()
                      .Request()
                      .PostAsync();
                Console.WriteLine("Accepted the event invite");
                    
            }
            catch (Exception error)
            {

                Console.WriteLine(error.Message);
            }
        }

		
        public async Task<IUserEventsCollectionPage> GetEvents()
        {
            return await graphClient
                .Me
                .Events
                .Request()
                .GetAsync();
        }
```

2. Add the **accept-event** command to the list of available commands in the **main** function
```csharp
 "\t 4. accept-event \n " + 
```

3. Add the following **case** statement in the **runAsync** method
```csharp
	case "accept-event":
		var eventsToAccept = await cal.GetEvents();

		for (int i = 0; i < 5; i++)
		{
			Event eventToAccept = eventsToAccept[i];

			Console.WriteLine($"Index: {i} Organiser: {eventToAccept.Organizer.EmailAddress.Name} Subject: {eventToAccept.Subject}");
		}

		Console.WriteLine("\nEnter the index of the invite you wish to accept");
		var indexToAccept = int.Parse(Console.ReadLine());

		await cal.AcceptAsync(eventsToAccept[indexToAccept].Id);
		Console.WriteLine("Invite accepted!");
		break;
```

## Exercise 10: Decline an invite to an event
In this exercise you are going to decline an invite to an event.

1. Add the code below to **CalendarController.cs**
```csharp
        /// <summary>
        /// Declines an invite to an event
        /// </summary>
        /// <param name="eventId"></param>
        /// <returns></returns>
        public async Task DeclineAsync(string eventId)
        {
            try
            {
                await graphClient
                    .Me
                    .Events[eventId]
                    .Decline()
                    .Request()
                    .PostAsync();
                Console.WriteLine("Event declined");

            } 
            catch (Exception error)
            {
                Console.WriteLine(error.Message);
            }
        }
```

2. Add the **decline-event** command to the list of available commands in the **main** function
```csharp
 "\t 4. decline \n " + 
```

3. Add the following **case** statement in the **runAsync** method
```csharp
	case "decline-event":
		var eventsToDecline = await cal.GetEvents();

		for (int i = 0; i < 5; i++)
		{
			Event eventToDecline = eventsToDecline[i];

			Console.WriteLine($"Index: {i} Organiser: {eventToDecline.Organizer.EmailAddress.Name} Subject: {eventToDecline.Subject}");
		}

		Console.WriteLine("\nEnter the index of the invite you wish to accept");
		var indexToDecline = int.Parse(Console.ReadLine());

		await cal.AcceptAsync(eventsToDecline[indexToDecline].Id);
		Console.WriteLine("Invite declined!");
		break;
```
