# DTO (Data Transfer Objects) та Validation

## Суть підходу

Ніколи не прокидайте об'єкти БД (Entities) напряму у відповідь API. Замість цього використовуються окремі DTO (Data Transfer Objects) з декораторами `class-validator`.

## Чому не можна віддавати Entity напряму

- **Безпека:** Entity може містити поля, які не мають потрапляти назовні (хеш пароля, внутрішні службові поля, зв'язки з іншими таблицями).
- **Стабільність контракту:** Зміна структури таблиці в БД не повинна автоматично ламати відповідь API.
- **Валідація на вході:** без окремого DTO немає чіткого місця, де перевіряти, що саме прийшло від клієнта.

## Приклад

```ts
// DTO для вхідних даних — з валідацією
export class CreateUserDto {
  @IsString()
  @MinLength(2)
  name: string;

  @IsEmail()
  email: string;

  @IsString()
  @MinLength(8)
  password: string;
}
```

```ts
// Entity — внутрішнє представлення в БД, не для видачі назовні
@Entity()
export class UserEntity {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  name: string;

  @Column()
  email: string;

  @Column()
  passwordHash: string; // ніколи не повинно потрапити у відповідь API
}
```

```ts
// DTO для відповіді — тільки те, що безпечно віддавати клієнту
export class UserResponseDto {
  id: number;
  name: string;
  email: string;
}
```

```ts
@Controller('users')
export class UserController {
  constructor(private readonly userService: UserService) {}

  @Post()
  async create(@Body() dto: CreateUserDto): Promise<UserResponseDto> {
    const user = await this.userService.create(dto);
    return { id: user.id, name: user.name, email: user.email };
  }
}
```

Nest автоматично застосовує валідацію через `ValidationPipe` (глобально або на рівні контролера), спираючись саме на декоратори з `class-validator` у DTO:

```ts
app.useGlobalPipes(new ValidationPipe({ whitelist: true }));
```

Опція `whitelist: true` додатково відкидає всі поля, яких немає в DTO — це страхує від того, що клієнт передасть зайві/небезпечні поля.

Такий підхід гарантує чистоту даних на вході (валідація) і безпеку на виході (жодних зайвих полів Entity в відповіді).

<!-- Місце для доповнення: class-transformer (@Expose/@Exclude), PartialType для Update DTO, вкладені DTO та @ValidateNested -->
