# SBT starting point

Being good at coding, and being good at build configuration are orthogonal skillsets. It's inherently a complex domain,
and typically for newcomers to coding, it's overwhelming to learn how to configure a build and what you're even trying
to accomplish - it's a huge domain. `sbt` has a reputation for being complex, and it's not uncommon for newcomers to
get lost in the weeds. I don't actually think `sbt` is adding complexity, it's just not hiding the inherent complexity
of the world of build configuration. We also have a tendency of splitting up our build into multiple files preemptively,
which obfuscates the overall build, asks people to keep multiple files in their head at a time.

Your build.sbt file should therefore be *dogmatically* well-documented with scaladocs and inline comments - prioritized
for legibility and clarity; it is paramount that the build.sbt file is *self-documenting*, and that it is *easy to
understand* for new team members. If you are doing _anything_ complex, or that you had to look up - add a comment with
both what and why it is being achieved.

Limit your usage of plugins to those that are absolutely necessary to accomplish your tasks; the more complex
your build, the more likely you are to run into issues with plugins when updating/upgrading, and the less legible
it will become.

## Core Build Philosophy

- This guide pushes toward a single-file approach that may not work well for all teams or projects; make an active
  choice by thinking through your reasons for deviating - is it by habitual convention, or do you actually find it
  helpful? Nothing here is dogma, but it is a "good starting point".
- Reduce cognitive overhead by being explicit, consistent, and consistent with the rest of the team. If there is any
  areas of interest (surprising compiler flags, custom tasks, etc) - document both what and why. Make it maintainable
  for other people than yourself.
- Prefer `.sbt` files over `.scala` build definitions unless programmatic logic is absolutely required
- Minimize file-sprawl - fewer build files are better.
- Prefer built-in SBT features over custom tasks and settings
- Prioritize readability and simplicity over complex abstractions
- Reduce cognitive overhead for new team members

## On "Complexity"

- Build complexity is likely to scale with the size of the project. This is inherently unavoidable as more
  functionality == more lines of code. But deviations from "the simple"
  should be documented. If you find yourself splitting the build into separate files, it should be documented in the
  main `build.sbt`, and you should be sure to explain why you're doing this; it will help both yourself and future
  newcomers to the project.
-

## Deliberate break with convention of splitting into separate files

- Much of the SBT/Scala ecosystem recommends splitting out Resolvers and Dependencies into separate files in `./project`
  I don't consider it wrong, but I have found it to be unnecessary for a small project below ~10 modules.
- I've found it more ergonomic to my workflow to avoid having to jump between files just to bump/remove a
  version dependency; there's very little
  confusion to be had when Dependencies are visible within the build.sbt file. It also makes sharing it (in
  slack, in stack overflow posts, with your LLM of choice) easier since you get the entire context presented to you. If
  you're a long-time SBT user... try it out, you might like it.

## Project Structure Rules

### REQUIRED File Structure:

```
project/
├── build.properties          # SBT version specification
├── plugins.sbt              # Plugin dependencies ONLY
build.sbt                    # Main build definition - ALL other config goes here
```

### Avoid:

- `project/Dependencies.scala` - Keep dependencies in build.sbt, it's ultimately just a list of dependencies, and you're
  not extracting logic out by using a separate file.
- Multiple `.sbt` files unless absolutely necessary

## build.sbt Organization:

Structure build.sbt in this order (I recommend keeping the with section separators:)

```scala
// =============================================================================
// Build Configuration
// =============================================================================
// Global settings, compiler options, build-wide configuration

// =============================================================================
// Project Definitions
// =============================================================================
// All lazy val project definitions

// =============================================================================
// Command Aliases
// =============================================================================
// Custom command shortcuts

// =============================================================================
// Dependencies
// =============================================================================
// Dependency object and all library dependencies
```

## Settings Scope Rules

### Use These Scopes Correctly:

- **Global**: ONLY for build source reloading and similar build-level concerns
- **ThisBuild**: For settings that apply to ALL projects (scalaVersion, organization, scalacOptions)
- **Project-specific**: For settings unique to a single project

### Standard ThisBuild Settings:

```scala
Global / onChangedBuildSource := ReloadOnSourceChanges
ThisBuild / scalaVersion      := "3.3.6"  // Use - and stay up to date with - latest patch of the Long Term Stable (LTS) version unless you have a reason not to
ThisBuild / organization      := "com.yourorg"
```

## Compiler Options

Define compiler options if required. Most mature projects will also add https://github.com/typelevel/sbt-tpolecat for
more strict/standard compiler options;
if you're just setting up, a reasonable starting point is may look like:

```scala
ThisBuild / scalacOptions ++= Seq(
  "-rewrite",                  // Automatically rewrite what compiler can, when code is compiled.
  "-no-indent",                // Enforce braces instead of indentation-based syntax (or specify `-indent` if you like it - just be consistent throughout your entire codebase)
  "-Wunused:all",              // Warn about unused imports, variables, etc.
  "-Wvalue-discard",           // Warn when non-Unit values are discarded
  "-deprecation",              // Show deprecation warnings
  "-feature"                   // Show feature warnings
)
```

## Dependencies structure in build.sbt

Use this pattern for dependency management:

```scala
lazy val Dependencies = new {
  
  // Private version object - ALWAYS use this pattern
  private object V {
    val cats       = "2.13.0"
    val catsEffect = "3.6.3"
    val circe      = "0.14.14"
    // Add all version constants here
  }

  // Logical groupings - use these standard names
  val service = Seq(
    "org.typelevel" %% "cats-core"   % V.cats,
    "org.typelevel" %% "cats-effect" % V.catsEffect
  )

  val test = Seq(
    "org.scalameta" %% "munit"              % V.munit % Test,
    "org.typelevel" %% "munit-cats-effect"  % V.munitCE % Test
  )
  
  // Additional standard groupings: tools, canaryTest, integration
}
```

### Dependency Rules:

- Group dependencies logically: `service`, `test`, `tools`, `canaryTest`, `integration`
- ALL versions go in private `V` object
- Document version constraints with `// comments`
- Use `.cross(CrossVersion.for3Use2_13)` for Scala 2.13 libraries when needed

## Project Definition Rules

### Single Module Project:

```scala
lazy val root = (project in file("."))
  .settings(
    name := "project-name",
    libraryDependencies ++= Dependencies.service ++ Dependencies.test
  )
```

### Multi-Module Project Pattern:

```scala
lazy val root = (project in file("."))
  .aggregate(service, tests, tools)  // List all modules
  .settings(name := "project-root")

lazy val service = (project in file("service"))
  .settings(
    name := "service-name",
    libraryDependencies ++= Dependencies.service ++ Dependencies.test
  )

lazy val tests = (project in file("tests"))
  .dependsOn(service % "test->test;compile->compile") // to make the contents from `service` available to `tests` 
  .settings(
    name := "service-tests",
    Test / fork := false
  )
```

## Plugin Management Rules

### plugins.sbt:

A common set of plugins to start from:

```scala
addSbtPlugin("com.github.sbt" % "sbt-native-packager" % "1.9.16")
addSbtPlugin("org.scalameta"  % "sbt-scalafmt"        % "2.5.2")
addSbtPlugin("ch.epfl.scala"  % "sbt-scalafix"        % "0.11.1")
```

## Command Aliases - STANDARD SET

Always include these command aliases:

```scala
addCommandAlias("commitCheck", "clean;compile;scalafmtAll;test")
addCommandAlias("cc", "commitCheck")
```

## Code Generation Patterns

### Version File Generation (when needed):

```scala
Compile / resourceGenerators += Def.task {
  val file = (Compile / resourceManaged).value / "version.properties"
  val content = s"version=${version.value}\ncommit=${git.gitHeadCommit.value.getOrElse("unknown")}"
  IO.write(file, content)
  Seq(file)
}
```

## JVM Configuration

### Development JVM Options:

```scala
ThisBuild / run / javaOptions := Seq(
  "-Dlogback.configurationFile=local-logback.xml",
  "--add-opens", "java.base/java.lang.invoke=ALL-UNNAMED"  // For JVM 11+
)
```

## ANTI-PATTERNS - AVOID THESE

1. ❌ Dependencies in `project/Dependencies.scala`
2. ❌ Using Global scope for regular project settings
3. ❌ Complex custom tasks for simple operations
4. ❌ Multiple `.sbt` files without strong justification
5. ❌ Mixing plugin configuration with dependency management
6. ❌ Hardcoded version strings in dependency declarations
7. ❌ Complex build logic for simple projects

## Required Comments and Documentation

- ALWAYS use section separators with `// =============================================================================`
- Document compiler flags and their purposes
- Add comments for version constraints and compatibility requirements

## Template Application Order

When creating SBT projects:

1. Set up file structure (project/, build.sbt)
2. Add section separators and basic ThisBuild settings
3. Define project structure with proper aggregation
4. Add command aliases
5. Create Dependencies object with proper structure
6. Wire up dependencies to projects
7. Add plugins only where needed
8. Configure testing and JVM options

## Scala Version Strategy

- Use latest stable Scala 3 LTS version as default
- Document any version constraints for libraries in `// comments`
- Prefer Scala 3 native libraries over cross-compiled ones when available