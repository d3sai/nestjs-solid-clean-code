# DTOs (Data Transfer Objects) and Validation

## Core Idea

Never expose database objects (Entities) directly in an API response. Use separate DTOs (Data Transfer Objects) with `class-validator` decorators instead.

## Why You Can't Return an Entity Directly

- **Security:** An Entity may contain fields that should never be exposed externally (password hash, internal service fields, relations to other tables).
- **Contract stability:** A change to the database table structure shouldn't automatically break the API response.
- **Input validation:** without a separate DTO, there's no clear place to check exactly what the client sent.

## Example

```ts
// DTO for input data — with validation
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
// Entity — internal DB representation, not for external exposure
@Entity()
export class UserEntity {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  name: string;

  @Column()
  email: string;

  @Column()
  passwordHash: string; // must never end up in an API response
}
```

```ts
// DTO for the response — only what's safe to return to the client
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

Nest automatically applies validation through `ValidationPipe` (globally or at the controller level), relying precisely on the `class-validator` decorators in the DTO:

```ts
app.useGlobalPipes(new ValidationPipe({ whitelist: true }));
```

The `whitelist: true` option additionally strips out any fields not present in the DTO — this guards against the client sending extra or unsafe fields.

This approach guarantees clean data on input (validation) and safety on output (no stray Entity fields in the response).

<!-- Open for expansion: class-transformer (@Expose/@Exclude), PartialType for Update DTOs, nested DTOs and @ValidateNested -->
