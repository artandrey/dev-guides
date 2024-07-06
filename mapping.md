# Mapping Guide

## Intro

This guide covers several ways to transform one data type to another. A common example is when we receive data in the form of a DTO (Data Transfer Object) from our API and then want to process this data in our application (perform some business logic using this data).

## Task

We are building a calendar app where users will be able to create reminders for different events.

**User Story**:
As a user, I want to create a reminder so that I can remember important events.

## Implementation

First of all, let's define a model of a reminder:

```tsx
class Reminder {
    public id: string;
    public title: string;
    public description: string;
    public eventDate: Date;
}
```

This model is used in `CalendarService`, for example, to add a reminder to the calendar:

```tsx
class CalendarService {
    addReminder(reminder: Reminder): Promise<Reminder> {
        // save reminder to db and return saved reminder
    }
}
```

Since we are building a backend app, we want the user to send us JSON:

```json
{
    "title": "Dan's birthday",
    "description": "I always forget about Dan's birthday so he always gets angry with me",
    "eventDate": "04-05-2025"
}
```

To accept data in this form, we will define it as a DTO:

```tsx
class CreateReminderDto {
    public title: string;
    public description: string;
    public eventDate: string;
}
```

Finally, here is a controller that handles requests:

```tsx
class CalendarController {
    constructor(
        private readonly calendarService: CalendarService,
        private readonly reminderMapper: ReminderMapper
    );

    public async postReminder(req: Request, res: Response) {
        const reminderDto: CreateReminderDto = req.body;
        const reminderModel = reminderMapper.toModel(reminderDto);
        await calendarService.addReminder(reminderModel);
        res.status(204).end();
    }
}
```

Don't forget that our business logic works with the model, so we need to map the DTO to the model:

**Mapper v1**

```tsx
class ReminderMapper {
    toModel(reminderDto: CreateReminderDto): Reminder {
        return {
            id: undefined,
            title: reminderDto.title,
            description: reminderDto.description,
            eventDate: new Date(reminderDto.eventDate),
        };
    }
}
```

As you know, if we construct an object that satisfies an interface or class, we can use that object where that interface or class is expected, even though there was no declarative relationship between the two. That is how our mapper works.

But then, we decided to add some logic to the model:

```tsx
class Reminder {
    public id: string;
    public title: string;
    public description: string;
    public eventDate: Date;

    // check if the reminder belongs to an event in the past or not
    public isPassed() {
        return this.eventDate < new Date();
    }
}
```

And now we get an error in our mapper:

```tsx
class ReminderMapper {
    toModel(reminderDto: CreateReminderDto): Reminder {
        return {
            id: undefined,
            title: reminderDto.title,
            description: reminderDto.description,
            eventDate: new Date(reminderDto.eventDate),
            // Error: Property 'isPassed' is missing in type '{...}' but required in type 'Reminder'.
        };
    }
}
```

That's because our object signature does not include the `isPassed` property in it.

Of course, to resolve this, we may just use the constructor of the class, like so:

```tsx
class Reminder {
    constructor(
        public title: string,
        public description: string,
        public eventDate: Date,
        public id?: string
    ) {}

    public isPassed() {
        return this.eventDate < new Date();
    }
}
```

and map it like this:
**Mapper v2**

```tsx
class ReminderMapper {
    toModel(reminderDto: CreateReminderDto): Reminder {
        return new Reminder(
            reminderDto.title,
            reminderDto.description,
            new Date(reminderDto.eventDate)
        );
    }
}
```

That looks nice until we get more properties for the Reminder.

What if the Reminder class grows and has 21 properties?

```tsx
class Reminder {
    constructor(
        public title: string,
        public description: string,
        public eventDate: Date,
        // 17 more properties ...
        public id?: string
    ) {}

    public isPassed() {
        return this.eventDate < new Date();
    }
}
```

Now updating the mapper seems like a task for half an hour because we need to make sure that the sequence of arguments in the constructor is correct and we did not shuffle the description and title. Not good for us.

Definitely, using a `builder` here will be a much better decision.

But I would suggest my way of mapping things, without the need to implement a builder or other additional stuff.

**Mapper v3**

```tsx
import { plainToInstance } from 'class-transformer';

class ReminderMapper {
    toModel(reminderDto: CreateReminderDto): Reminder {
        return plainToInstance(Reminder, reminderDto);
    }
}
```

In this case, we use a library called `class-transformer` to transform the `reminderDto` object into a `Reminder` instance.

```tsx
const reminderMapper = new ReminderMapper();

const reminderModel = reminderMapper.toModel({
    title: 'Test',
    description: 'Description',
    eventDate: '20-04-2024',
});
reminderModel.isPassed(); // true
```

As we see, we have saved all class functionality, and that's exactly what we want. Also, notice that we don't even need to transform the string to a date because it was already done by `class-transformer`.

But what if we change `CreateReminderDto`?

```tsx
class CreateReminderDto {
    public title: string;
    public details: string; // formerly description
    public date: string; // formerly eventDate
}
```

Now we will loose some fields from our DTO in mapped model:

```tsx
const reminderMapper = new ReminderMapper();

const reminderModel = reminderMapper.toModel({
    title: 'Test',
    details: 'Description',
    date: '20-04-2024',
});
reminderModel.description; // undefined
```

`reminderModel.description` is `undefined` because `class-transformer` is not aware that it needs to map `details` to `description` now.

To resolve this issue, we may manually define the object shape in our mapper:

```tsx
import { plainToInstance } from 'class-transformer';

class ReminderMapper {
    toModel(reminderDto: CreateReminderDto): Reminder {
        return plainToInstance(Reminder, {
            ...reminderDto,
            description: reminderDto.details,
            eventDate: reminderDto.date,
        });
    }
}
```

To enable autocomplete, you may also use the `satisfies` keyword:

```tsx
import { plainToInstance } from 'class-transformer';

class ReminderMapper {
    toModel(reminderDto: CreateReminderDto): Reminder {
        return plainToInstance(Reminder, {
            ...reminderDto,
            description: reminderDto.details,
            eventDate: reminderDto.date,
        } satisfies Partial<Reminder>);
    }
}
```
