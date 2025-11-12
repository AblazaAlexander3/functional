# Functional coding

## LanguageExt for C# Developers — A Hands‑On Training (Full Details + Code)
### Table of Contents

### why-languageext-in-c-terms
#### setup--project-template
#### module-1--core-fp-mental-model
#### module-2--options-eithers-validations-try
#### module-3--immutable-collections
#### module-4--effects--async-effaff-vs-trytryasync
#### module-5--reader-for-dependency-injection
#### module-6--validation--error-modeling-for-security
#### module-7--interop-with-tasks-exceptions-and-linq
#### module-8--testing-xunit--property-based-hints
#### module-9--patterns--anti-patterns
#### module-10--cheatsheet
#### capstone-mini-project-exercise--solution

## Why LanguageExt (in C# terms)

Null-safety without null → Use Option<T> instead of T?.
Typed errors instead of exceptions → Use Either<L, R> or Validation<Fail, A>.
Predictable side effects → Model them as Eff<T> (sync) or Aff<T> (async).
Immutable by default → Persistent data structures like Lst<T>, Map<K,V>.
Composability → Use Map, Bind (Select, SelectMany) and pattern matching (Match) to compose logic.
Testable → Pure functions with explicit effects are easier to unit test.

If you’ve used JS Promises/monads (like in Ramda/folktale), the flow feels familiar—just strongly typed and integrated with C#.


## Setup & Project Template

### Install NuGet packages:

dotnet new console -n LanguageExtTraining
cd LanguageExtTraining
dotnet add package LanguageExt.Core
dotnet add package LanguageExt.Effects --version 4.*     # if you’ll use Eff/Aff
dotnet add package LanguageExt.Parsec --version 4.*       # optional, parser combinators
dotnet add package FluentAssertions
dotnet add package xunit



###Program.cs
using System;
using LanguageExt;
using static LanguageExt.Prelude;

public static class Program
{
    public static void Main()
    {
        // Verify the package is working
        Option<int> maybe = Some(42);
        Console.WriteLine(maybe.Match(
            Some: value => $"Value: {value}",
            None: () => "No value"
        ));
    }
}


#Tip: Prefer using static LanguageExt.Prelude; for Some, None, Right, Left, Try, etc.


Module 1 — Core FP Mental Model


Key ideas:
	• Expressions over statements: Prefer returning a value over mutating state.
	• Total functions: Handle all inputs (e.g., avoid null, avoid unhandled exceptions).
	• Pure functions: No hidden I/O; pass dependencies explicitly or model them as effects.
	• Immutability: Prefer persistent data structures.
	
C# pattern matching + LanguageExt Match:

/// <summary>
/// Example: safe divide returning Option.
/// </summary>
public static Option<double> SafeDivide(double numerator, double denominator)
{
    return denominator == 0
        ? Option<double>.None
        : Some(numerator / denominator);
}
public static string DescribeOption(Option<double> value)
{
    return value.Match(
        Some: v => $"Result = {v:F2}",
        None: () => "No result"
    );
}


Module 2 — Options, Eithers, Validations, Try
Option<T> — absence without null

using LanguageExt;
using static LanguageExt.Prelude;
/// <summary>Try parse int with Option.</summary>
public static Option<int> ParseInt(string text)
{
    return int.TryParse(text, out int value) ? Some(value) : None;
}
public static Option<int> LookupUserAge(Map<string, int> ages, string username)
{
    return ages.Find(username); // returns Option<int>
}
public static Option<int> AddAges(Option<int> a, Option<int> b)
{
    // Use LINQ SelectMany (monadic bind)
    return
        from aa in a
        from bb in b
        select aa + bb;
}


Either<L, R> — typed error or success

/// <summary>
/// Domain error model.
/// </summary>
public abstract record AppError(string Message)
{
    public sealed record NotFound(string Resource) : AppError($"Not found: {Resource}");
    public sealed record InvalidInput(string Reason) : AppError($"Invalid: {Reason}");
    public sealed record Unauthorized : AppError("Unauthorized");
}
/// <summary>Find user; Left = error, Right = user.</summary>
public static Either<AppError, User> GetUserById(Map<int, User> db, int id)
{
    return db.Find(id)
             .ToEither<AppError>(new AppError.NotFound($"User {id}"));
}
public static string GreetUser(Either<AppError, User> user)
{
    return user.Match(
        Right: u => $"Hello, {u.FullName}.",
        Left:  e => $"Error: {e.Message}"
    );
}
public sealed record User(int Id, string FullName);


Validation<Fail, A> — accumulate errors (great for input validation)

using LanguageExt;
using static LanguageExt.Prelude;
using Fail = LanguageExt.Seq<string>; // or create a dedicated error type

/// <summary>
/// Validate user signup; accumulate all failures.
/// </summary>
public static Validation<Fail, Signup> ValidateSignup(string email, string password, int age)
{
    Validation<Fail, string> emailV =
        IsValidEmail(email) ? Success<Fail, string>(email) : Fail<Fail, string>(Seq1("Invalid email."));

    Validation<Fail, string> passwordV =
        password.Length >= 12 ? Success<Fail, string>(password) : Fail<Fail, string>(Seq1("Password too short."));

    Validation<Fail, int> ageV =
        age >= 18 ? Success<Fail, int>(age) : Fail<Fail, int>(Seq1("Must be at least 18."));

    // Applicative style: all checks run, errors accumulate
    return (emailV, passwordV, ageV).Apply((e, p, a) => new Signup(e, p, a));
}

public sealed record Signup(string Email, string Password, int Age);

private static bool IsValidEmail(string email) =>
    !string.IsNullOrWhiteSpace(email) && email.Contains('@');



Try<T> / TryAsync<T> — wrap exception-throwing code


/// <summary>
/// Try reading env var as int safely using Try.
/// </summary>
public static Try<int> GetEnvPort(string name) => Try(() =>
{
    string? value = Environment.GetEnvironmentVariable(name);
    if (string.IsNullOrWhiteSpace(value))
        throw new InvalidOperationException($"Missing environment variable: {name}");
    return int.Parse(value);
});
// Usage:
public static string DescribeTry(Try<int> t) =>
    t.Match(
        Succ: v => $"Port: {v}",
        Fail: ex => $"Failed: {ex.Message}"
    );

Module 3 — Immutable Collections
LanguageExt provides persistent collections: Lst<T>, Arr<T>, Seq<T>, Set<T>, HashSet<T>, Map<K,V>, HashMap<K,V>.
	• Arr<T>: fast index lookup (array-like), persistent.
	• Lst<T>: linked-list semantics, cheap head/tail.
	• Map<K,V>: ordered map; HashMap<K,V>: hash-based map.
	• Interoperability: Convert to/from IEnumerable<T> easily.



public static Map<string, int> BuildAgeMap()
{
    Map<string, int> ages = Map(("alex", 33), ("marie", 28));
    ages = ages.AddOrUpdate("marie", 29);
    return ages;
}
public static int SumAges(Lst<int> ages) =>




Module 4 — Effects & Async (Eff/Aff) vs Try/TryAsync
	• Try/TryAsync wrap impure code that might throw (pulling exceptions into values).
	• Eff<T> / Aff<T> model effects that can fail with a Error and are deferred until run. 
		○ Eff<T> = synchronous effect.
		○ Aff<T> = asynchronous effect.
	• They typically return Fin<T> when executed (.Run() / .Run() async), which is success or error.


using LanguageExt.Effects.Traits;
using LanguageExt;
using static LanguageExt.Prelude;
/// <summary>Read a file as an effect (sync).</summary>
public static Eff<string> ReadFileEff(string path) => Eff(() =>
{
    return System.IO.File.ReadAllText(path);
});
/// <summary>Call HTTP API as an async effect (sketched).</summary>
public static Aff<string> GetHttpAff(Uri uri) => Aff(async () =>
{
    using var client = new System.Net.Http.HttpClient();
    string json = await client.GetStringAsync(uri);
    return json;
});
// Composing effects:
public static Aff<int> GetConfigPortThenCall(Uri uri, string envName) =>
    from portStr in Aff(async () => Environment.GetEnvironmentVariable(envName) ?? "8080")
    from _ in RightAff(unit) // sequencing example
    from body in GetHttpAff(new Uri($"{uri}?port={portStr}"))
    select body.Length;


Running effect:
public static async Task<string> RunEffectDemo()
{
    Fin<string> fileFin = ReadFileEff("config.json").Run();
    string fileResult = fileFin.Match(
        Succ: text => $"Read {text.Length} chars",
        Fail: err  => $"Error: {err.Message}"
    );
Fin<int> httpFin = await GetConfigPortThenCall(new Uri("https://example.com/api"), "APP_PORT").Run();
    string httpResult = httpFin.Match(
        Succ: n => $"Response length: {n}",
        Fail: err => $"HTTP error: {err.Message}"
    );
return $"{fileResult}; {httpResult}";
}

When to use what
	• Prefer Either/Validation for domain success/failure.
	• Use Try/TryAsync to capture exceptions from impure APIs.
	• Use Eff/Aff to model side-effects that you want to compose and run at boundaries.




Module 5 — Reader for Dependency Injection
Reader<Env, A> lets you pass dependencies/configuration without mutable globals/DI containers.


using LanguageExt;
using static LanguageExt.Prelude;
public sealed record AppEnv(System.Net.Http.HttpClient Http, string BaseUrl);
public static Reader<AppEnv, Aff<string>> GetUserJson(int id) =>
    from env in ask<AppEnv>()
    select GetHttpAff(new Uri($"{env.BaseUrl}/users/{id}"));
public static async Task<Fin<string>> RunReaderDemo()
{
    var env = new AppEnv(new System.Net.Http.HttpClient(), "https://example.com/api");
    Reader<AppEnv, Aff<string>> prog = GetUserJson(123);
    // Provide env at the edge:
    Aff<string> effect = prog.Run(env);
    return await effect.Run();
}


This keeps your core logic purely declarative and testable—you can supply a mock AppEnv in tests.

Module 6 — Validation & Error Modeling for Security
Goals:
	• Never trust input. Validate early with Validation (accumulate).
	• Represent authorization/authentication errors with distinct types (for auditability).
	• Avoid throwing exceptions for expected business errors; encode them in the type.




public abstract record SecError(string Message)
{
    public sealed record AuthnFailed : SecError("Authentication failed.");
    public sealed record AuthzDenied(string Action) : SecError($"Not allowed: {Action}");
    public sealed record InputInvalid(Seq<string> Reasons) : SecError("Invalid input.");
}
// Domain result type
public readonly record struct Result<T>(Either<SecError, T> Value)
{
    public static Result<T> Ok(T value) => new(Right<SecError, T>(value));
    public static Result<T> Fail(SecError error) => new(Left<SecError, T>(error));
}
public static Result<User> EnsureAdult(Signup s)
{
    return s.Age >= 18
        ? Result<User>.Ok(new User(1, s.Email))
        : Result<User>.Fail(new SecError.AuthzDenied("Signup for minors"));
}
Module 7 — Interop with Tasks, Exceptions, and LINQ
From Task<T> to Aff<T>:


public static Aff<T> ToAff<T>(Func<Task<T>> task) => Aff(async () => await task());



From Task<T> to TryAsync<T>:


public static TryAsync<T> ToTryAsync<T>(Func<Task<T>> task) => TryAsync(async () => await task());


LINQ query syntax works with Option, Either, Try, Eff/Aff:


public static Either<AppError, string> Compose(
    Either<AppError, int> a,
    Either<AppError, int> b)
{
    return
        from aa in a
        from bb in b
        select $"Sum = {aa + bb}";
}

Module 8 — Testing (xUnit) & Property-Based Hints
Unit test a pure function:


using Xunit;
using FluentAssertions;
public class OptionTests
{
    [Fact]
    public void SafeDivide_ShouldReturnNone_OnDivideByZero()
    {
        Option<double> result = SafeDivide(10, 0);
        result.IsNone.Should().BeTrue();
    }
[Fact]
    public void SafeDivide_ShouldReturnSome_OnValidInput()
    {
        Option<double> result = SafeDivide(9, 3);
        result.IsSome.Should().BeTrue();
        result.IfNone(0).Should().Be(3);
    }
}
Testing effects: keep effects at edges; replace them with controlled effects (e.g., a fake Reader env or a deterministic Aff).
Property-based testing (optional): use FsCheck to assert algebraic properties (e.g., associativity of Bind, idempotence of sanitize functions).


Module 9 — Patterns & Anti-Patterns
Patterns
	• Define a single error algebra per bounded context (AppError, SecError) and reuse.
	• Keep functions pure; push effects to the edges using Eff/Aff or Reader.
	• Validate at boundaries with Validation, promote to Either for business flow.
	• Use LINQ query expressions to compose monadic flows for readable pipelines.
	• Prefer immutable collections; interop with IEnumerable<T> only when necessary.
Anti-Patterns
	• Throwing exceptions for expected domain errors (use Either/Validation).
	• Returning null from FP APIs (use Option).
	• Mutating global state in the middle of pipelines.
	• Deeply nesting Match—prefer composition (Map, Bind) and only Match at the edges (e.g., controller).


Module 10 — Cheatsheet.

// Create
Option<T> o = Some(value) / None;
Either<L,R> e = Right<R>(value) / Left<L>(error);
Validation<Fail,T> v = Success<T>(value) / Fail<Fail, T>(Seq("error"));
Try<T> t = Try(() => PossiblyThrowing());
// Transform
o.Map(f); e.Map(f); v.Map(f); t.Map(f);
// Chain
o.Bind(f); e.Bind(f); v.Bind(f); t.Bind(f);
// Query syntax
from x in monadA
from y in monadB
select f(x,y);
// Pattern match
o.Match(Some: x => ..., None: () => ...);
e.Match(Right: x => ..., Left: l => ...);
// Effects
Eff<T> eff = Eff(() => ...);
Aff<T> aff = Aff(async () => ...);
Fin<T> fin = await aff.Run(); // or eff.Run()

Capstone Mini‑Project (Exercise + Solution)
Scenario: Secure Signup Service
	• Requirements: 
		1. Validate input (email, password, age) and accumulate all errors.
		2. Check if email already exists (async repository).
		3. If valid and unique, create user and return Either<SecError, User>.
		4. No exceptions for expected flow; effects modeled with Aff.
Domain & Error Types



using LanguageExt;
using LanguageExt.Common;
using LanguageExt.Effects.Traits;
using static LanguageExt.Prelude;
public abstract record SecError(string Message)
{
    public sealed record InputInvalid(Seq<string> Reasons) : SecError("Invalid input.");
    public sealed record Conflict(string Field) : SecError($"Conflict on {Field}.");
    public sealed record Infra(string Detail) : SecError($"Infrastructure: {Detail}");
}
public sealed record Signup(string Email, string Password, int Age);
public sealed record User(int Id, string Email);
Validation (accumulate all errors)


public static Validation<Seq<string>, Signup> Validate(Signup s)
{
    Validation<Seq<string>, string> emailV =
        (!string.IsNullOrWhiteSpace(s.Email) && s.Email.Contains('@'))
            ? Success<Seq<string>, string>(s.Email)
            : Fail<Seq<string>, string>(Seq1("Email is invalid."));
Validation<Seq<string>, string> passV =
        s.Password.Length >= 12
            ? Success<Seq<string>, string>(s.Password)
            : Fail<Seq<string>, string>(Seq1("Password must be at least 12 chars."));
Validation<Seq<string>, int> ageV =
        s.Age >= 18
            ? Success<Seq<string>, int>(s.Age)
            : Fail<Seq<string>, int>(Seq1("Age must be >= 18."));
return (emailV, passV, ageV).Apply((e, p, a) => new Signup(e, p, a));
}


Repository (async effect)


public interface IUserRepo
{
    Task<bool> EmailExists(string email);
    Task<User> Insert(Signup s);
}
public sealed class InMemoryUserRepo : IUserRepo
{
    private readonly object _lock = new();
    private readonly Map<string, User> _db = Map<string, User>();
public Task<bool> EmailExists(string email)
    {
        bool exists = _db.ContainsKey(email.ToLowerInvariant());
        return Task.FromResult(exists);
    }
public Task<User> Insert(Signup s)
    {
        lock (_lock)
        {
            int id = _db.Count + 1;
            User user = new(id, s.Email);
            _db = _db.AddOrUpdate(s.Email.ToLowerInvariant(), user);
            return Task.FromResult(user);
        }
    }
}


(Note: Map is immutable; above uses a private field for simplicity. In real code, consider returning a new repo with updated state or encapsulating state mutation in effects.)
Service using Aff and Either


public sealed record AppEnv(IUserRepo Repo);
public static Reader<AppEnv, Aff<Either<SecError, User>>> SignupService(Signup raw)
{
    // 1) Validate (accumulate); if invalid, lift to Either Left
    Either<SecError, Signup> validated =
        Validate(raw).Match<Either<SecError, Signup>>(
            Succ: s => Right<SecError, Signup>(s),
            Fail: errs => Left<SecError, Signup>(new SecError.InputInvalid(errs))
        );
// 2) Compose async repository checks & insert as Aff
    return from env in ask<AppEnv>()
           select
               from s in Aff<Either<SecError, Signup>>(() => Task.FromResult(validated))
               from isTaken in Aff(async () => await env.Repo.EmailExists(s.Email))
               from user in isTaken
                    ? FailAff<Either<SecError, User>>(Error.New("conflict"))
                          .Map<Either<SecError, User>>(_ => throw new InvalidOperationException()) // not reached
                          .Catch(_ => Right<SecError, User>(new SecError.Conflict("email")))
                    : ToAff(async () => await env.Repo.Insert(s))
                          .Map<Either<SecError, User>>(u => Right<SecError, User>(u))
                          .Catch(ex => Right<SecError, User>(new SecError.Infra(ex.Message)))
               select user;
}
// Small helper to convert Task<T> to Aff<T>
public static Aff<T> ToAff<T>(Func<Task<T>> task) => Aff(async () => await task());

Running the flow

public static async Task<string> RunSignupDemo()
{
    var env = new AppEnv(new InMemoryUserRepo());
var attempt1 = await SignupService(new Signup("alex@example.com", "VeryStrongPass!", 25))
        .Run(env)   // Reader -> Aff
        .Run();     // Aff -> Fin<Either<SecError, User>>
var msg1 = attempt1.Match(
        Succ: res => res.Match(
            Right: u => $"Created user #{u.Id} ({u.Email})",
            Left:  e => $"Left (domain): {e.Message}"
        ),
        Fail: err => $"Aff failed: {err.Message}"
    );
var attempt2 = await SignupService(new Signup("alex@example.com", "Short", 15))
        .Run(env).Run();
var msg2 = attempt2.Match(
        Succ: res => res.Match(
            Right: u => $"Created user #{u.Id} ({u.Email})",
            Left:  e => e switch
            {
                SecError.InputInvalid inval => $"Invalid: {string.Join("; ", inval.Reasons)}",
                SecError.Conflict c => $"Conflict: {c.Field}",
                _ => $"Domain error: {e.Message}"
            }
        ),
        Fail: err => $"Aff failed: {err.Message}"
    );
return $"{msg1}\n{msg2}";
}


Show more lines
What you get:
	• All input errors shown at once.
	• Clear separation of domain errors vs infrastructure issues.
	• Effects are composed and only run at the edge, which is test-friendly.

Suggested Exercises (with Solutions Hints)
	1. Option Practice
		○ Implement Option<decimal> SafeSqrt(decimal); chain with SafeDivide.
		○ Hint: from x in SafeSqrt(a) from y in SafeDivide(x,b) select ....
	2. Either Practice
		○ Build a function Either<AppError, Token> Authenticate(Credentials) that never throws.
	3. Validation Practice
		○ Add rules: password must include upper, lower, digit, symbol; accumulate all failures.
	4. Eff/Aff Practice
		○ Wrap filesystem reads/writes in Eff/Aff and propagate a SecError.Infra on failures.
	5. Reader DI
		○ Add caching to AppEnv and compose it in SignupService without changing the core logic.

Style Notes for You
	• Explicit types: The samples use explicit Option<int>, Either<AppError, User>, etc.
	• XML docs: Use <summary> and describe error variants and return shapes.
	• SOLID/DRY: Keep error types reusable; separate pure validation from effectful infra; keep functions small and composable.
