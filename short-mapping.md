# Mapping using `class-transformer`

Imagine that we are building a calendar app and have a database schema for the reminder entity.

**Schema:**

```tsx
import { pgTable, uuid, text, timestamp } from 'drizzle-orm/pg-core';

export const reminders = pgTable('reminders', {
    id: uuid('id').primaryKey(),
    title: text('title').notNull(),
    description: text('description').notNull(),
    eventDate: timestamp('event_date').notNull(),
});
```

**Entity:**

```tsx
class Reminder {
    public id: string;
    public title: string;
    public description: string;
    public eventDate: Date;
}
```

**Repository:**

```tsx
class ReminderRepository {
    async findById(id: string): Promise<Reminder | null> {
        const result = await db.query.reminders.findFirst({
            where: (reminders, { eq }) => eq(reminders.id, id),
        });
        return result;
    }
}
```

Everything works well until we add some logic to our entity:

```tsx
class Reminder {
    public id: string;
    public title: string;
    public description: string;
    public eventDate: Date;

    // check if the reminder belongs to an event in the future or not
    public isPassed() {
        return this.eventDate < new Date();
    }
}
```

Then we will get an error in our repository because the Drizzle return type does not fully satisfy the Reminder type.

To resolve this issue, I propose using the `class-transformer` library to convert a plain object to a class:

**Repository:**

```tsx
import { plainToInstance } from 'class-transformer';

class ReminderRepository {
    async findById(id: string): Promise<Reminder | null> {
        const result = await db.query.reminders.findFirst({
            where: (reminders, { eq }) => eq(reminders.id, id),
        });
        if (!result) return null; // manually handling null case
        return plainToInstance(Reminder, result); // transforming
    }
}
```

Now we have access to the method of `Reminder`:

```tsx
class ReminderService {
    constructor(private readonly reminderRepository: ReminderRepository) {}

    public async getById(id: string) {
        const reminder = await reminderRepository.findById(id);
        reminder.isPassed(); // true
        // other logic...
    }
}
```
