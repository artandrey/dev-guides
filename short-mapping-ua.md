# Мапінг за допомогою `class-transformer`

Уявімо, що ми створюємо календарний додаток і маємо схему бази даних для сутності нагадування.

**Схема:**

```tsx
import { pgTable, uuid, text, timestamp } from 'drizzle-orm/pg-core';

export const reminders = pgTable('reminders', {
    id: uuid('id').primaryKey(),
    title: text('title').notNull(),
    description: text('description').notNull(),
    eventDate: timestamp('event_date').notNull(),
});
```

**Сутність:**

```tsx
class Reminder {
    public id: string;
    public title: string;
    public description: string;
    public eventDate: Date;
}
```

**Репозиторій:**

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

Все працює добре, доки ми не додаємо логіку до нашої сутності:

```tsx
class Reminder {
    public id: string;
    public title: string;
    public description: string;
    public eventDate: Date;

    // перевірка, чи нагадування відноситься до події в майбутньому чи минулому
    public isPassed() {
        return this.eventDate < new Date();
    }
}
```

Тоді ми отримаємо помилку у нашому репозиторії, оскільки тип, що повертається Drizzle, не повністю задовольняє тип Reminder.

Для вирішення цієї проблеми я пропоную використовувати бібліотеку `class-transformer` для перетворення plain-об'єкта на клас:

**Репозиторій:**

```tsx
import { plainToInstance } from 'class-transformer';

class ReminderRepository {
    async findById(id: string): Promise<Reminder | null> {
        const result = await db.query.reminders.findFirst({
            where: (reminders, { eq }) => eq(reminders.id, id),
        });
        if (!result) return null; // ручне оброблення випадку null
        return plainToInstance(Reminder, result); // перетворення
    }
}
```

Тепер ми маємо доступ до методу `Reminder`:

```tsx
class ReminderService {
    constructor(private readonly reminderRepository: ReminderRepository) {}

    public async getById(id: string) {
        const reminder = await reminderRepository.findById(id);
        reminder.isPassed(); // true
        // інша логіка...
    }
}
```
