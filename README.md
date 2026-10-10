# Standalone CodeNarc Setup, Usage, and Teardown Guide

This guide provides an isolated, zero-footprint workflow for running **CodeNarc** (static analysis and linting for Groovy and Jenkins pipeline scripts) across any project or repository. It does not require installing Gradle/Maven globally, does not alter project configurations, and can be completely removed with a single command.

---

## 1. Prerequisites

* **Java JDK / JRE** 11 or higher installed (`java -version`).
* **curl** for downloading JAR binaries.

---

## 2. Setup (Isolated / Zero-Pollution)

CodeNarc and its required Groovy runtime dependencies are downloaded into an isolated temporary directory (`/tmp/codenarc-runner`).

### Step 2.1: Download CodeNarc & Runtime Dependencies

```bash
# Create isolated working directory outside the repository
mkdir -p /tmp/codenarc-runner && cd /tmp/codenarc-runner

# Download CodeNarc and compatible Groovy 3.x runtime libraries
curl -sLO https://repo1.maven.org/maven2/org/codenarc/CodeNarc/3.5.0/CodeNarc-3.5.0.jar
curl -sLO https://repo1.maven.org/maven2/org/codehaus/groovy/groovy/3.0.22/groovy-3.0.22.jar
curl -sLO https://repo1.maven.org/maven2/org/codehaus/groovy/groovy-xml/3.0.22/groovy-xml-3.0.22.jar
curl -sLO https://repo1.maven.org/maven2/org/codehaus/groovy/groovy-templates/3.0.22/groovy-templates-3.0.22.jar
curl -sLO https://repo1.maven.org/maven2/org/slf4j/slf4j-api/2.0.13/slf4j-api-2.0.13.jar
curl -sLO https://repo1.maven.org/maven2/org/slf4j/slf4j-simple/2.0.13/slf4j-simple-2.0.13.jar
```

> **Compatibility Note (Groovy 3 vs 4):**
> CodeNarc 3.5.0 AST visitors are built against Groovy 3.0.x. Do not use Groovy 4.x binaries with CodeNarc 3.5.0, as changes in Groovy 4's AST visitor API will cause `MissingMethodException: visitClassEx` errors.

### Step 2.2: Create Base Ruleset (`/tmp/codenarc-runner/ruleset.groovy`)

Define a base ruleset in `/tmp/codenarc-runner/ruleset.groovy`. This default ruleset is tuned for Jenkins pipeline scripts, general Groovy code, variable scoping, and bug detection:

```bash
cat << 'RULESET_EOF' > /tmp/codenarc-runner/ruleset.groovy
ruleset {
    description 'Standalone CodeNarc Ruleset'

    // Jenkins CPS and serialization best practices
    ruleset('rulesets/jenkins.xml')

    // Fundamental bug detection and empty blocks
    ruleset('rulesets/basic.xml')

    // Unused variables, parameters, and fields
    ruleset('rulesets/unused.xml')

    // Import organization and unused imports
    ruleset('rulesets/imports.xml')

    // Variable scoping and script-binding checks (catches missing `def` keyword)
    ruleset('rulesets/concurrency.xml') {
        include 'NoScriptBindings'
    }
}
RULESET_EOF
```

> **Repository-Specific Rulesets:**
> If a target repository already contains its own ruleset (e.g. `config/codenarc.groovy`), substitute `-rulesetfiles="file:/path/to/repo/custom-ruleset.groovy"`.

---

## 3. How to Use CodeNarc (5 Step-by-Step Operations)

Run these commands from the root directory of the repository you want to inspect.

### Step 3.1: Output Violations to Console (stdout)
Scans target files and prints all violations, line numbers, and rule names directly to standard output:

```bash
java -cp "/tmp/codenarc-runner/*" org.codenarc.CodeNarc \
  -basedir="." \
  -includes="**/*.groovy,**/Jenkinsfile*,**/*Pipeline*,buildenv/jenkins/*" \
  -rulesetfiles="file:/tmp/codenarc-runner/ruleset.groovy" \
  -report=text:stdout
```

> **Scanning Extensionless Pipeline Scripts:**
> In many Jenkins repositories, pipeline scripts lack a `.groovy` extension (e.g. `JenkinsfileBase`, `openjdk_tests`, `testJobTemplate`, `jck_sync`). To ensure these files are analyzed, include their directory path in `-includes` (e.g. `buildenv/jenkins/*`) or specify them directly.

### Step 3.2: Generate and View an HTML Report
Generates a structured HTML report in `/tmp/codenarc-runner/`:

```bash
# Generate the report
java -cp "/tmp/codenarc-runner/*" org.codenarc.CodeNarc \
  -basedir="." \
  -includes="**/*.groovy,**/Jenkinsfile*,**/*Pipeline*,buildenv/jenkins/*" \
  -rulesetfiles="file:/tmp/codenarc-runner/ruleset.groovy" \
  -report="html:/tmp/codenarc-runner/CodeNarcReport.html"

# Open the report in your browser
xdg-open /tmp/codenarc-runner/CodeNarcReport.html
```

### Step 3.3: Target a Specific File or Subdirectory
Isolate analysis to a single file or directory to debug issues quickly:

```bash
# Analyze a specific directory (all Groovy + extensionless scripts)
java -cp "/tmp/codenarc-runner/*" org.codenarc.CodeNarc \
  -basedir="buildenv/jenkins" \
  -includes="**/*.groovy,openjdk_tests,JenkinsfileBase,testJobTemplate,getDependency,getTRSSOutput,jck_sync,perfL2JobTemplate" \
  -rulesetfiles="file:/tmp/codenarc-runner/ruleset.groovy" \
  -report=text:stdout

# Analyze a single file
java -cp "/tmp/codenarc-runner/*" org.codenarc.CodeNarc \
  -basedir="buildenv/jenkins" \
  -includes="JenkinsfileBase" \
  -rulesetfiles="file:/tmp/codenarc-runner/ruleset.groovy" \
  -report=text:stdout
```

> **Java 17+ JVM Compatibility:**
> When running on modern JDKs (Java 17, 21, 25), if reflection warnings or module access errors occur, pass the `--add-opens` JVM flag:
> ```bash
> java --add-opens java.base/java.lang=ALL-UNNAMED -cp "/tmp/codenarc-runner/*" org.codenarc.CodeNarc ...
> ```

### Step 3.4: Customize or Filter Active Rules
Edit `/tmp/codenarc-runner/ruleset.groovy` to add or exclude specific rules according to project needs:

```groovy
ruleset {
    // Enable complete rulesets
    ruleset('rulesets/jenkins.xml')
    ruleset('rulesets/basic.xml')
    ruleset('rulesets/unused.xml')
    ruleset('rulesets/imports.xml')

    // Selectively include specific rules from a category
    ruleset('rulesets/concurrency.xml') {
        include 'NoScriptBindings' // Catches variables assigned without def/type
    }

    // Exclude specific noisy rules from a ruleset
    ruleset('rulesets/unnecessary.xml') {
        exclude 'UnnecessaryGString'
        exclude 'UnnecessarySemicolon'
    }
}
```

### Step 3.5: Clean Up (Tear Down)
To completely remove all CodeNarc binaries, temporary rulesets, and generated reports:

```bash
rm -rf /tmp/codenarc-runner
```

---

### Step 3.6: Quick Ad-Hoc One-Liner (Setup, Scan, Teardown)
For an instant one-off scan of any repo without leaving background artifacts:

```bash
mkdir -p /tmp/cn && cd /tmp/cn && \
curl -sLO https://repo1.maven.org/maven2/org/codenarc/CodeNarc/3.5.0/CodeNarc-3.5.0.jar && \
curl -sLO https://repo1.maven.org/maven2/org/codehaus/groovy/groovy/3.0.22/groovy-3.0.22.jar && \
curl -sLO https://repo1.maven.org/maven2/org/codehaus/groovy/groovy-xml/3.0.22/groovy-xml-3.0.22.jar && \
curl -sLO https://repo1.maven.org/maven2/org/codehaus/groovy/groovy-templates/3.0.22/groovy-templates-3.0.22.jar && \
curl -sLO https://repo1.maven.org/maven2/org/slf4j/slf4j-api/2.0.13/slf4j-api-2.0.13.jar && \
curl -sLO https://repo1.maven.org/maven2/org/slf4j/slf4j-simple/2.0.13/slf4j-simple-2.0.13.jar && \
echo "ruleset { ruleset('rulesets/jenkins.xml'); ruleset('rulesets/basic.xml'); ruleset('rulesets/concurrency.xml') { include 'NoScriptBindings' } }" > ruleset.groovy && \
java -cp "/tmp/cn/*" org.codenarc.CodeNarc -basedir="$OLDPWD" -includes="**/*.groovy,**/Jenkinsfile*,buildenv/jenkins/*" -rulesetfiles="file:/tmp/cn/ruleset.groovy" -report=text:stdout; \
cd "$OLDPWD" && rm -rf /tmp/cn
```

---

## 4. Complementary Options for Testing Jenkins & Groovy Files Locally

Beyond standalone CodeNarc static analysis, here are the 5 comprehensive approaches for testing Jenkins pipelines and Groovy scripts locally:

### Option 1: Static Syntax & Linting (Without Live Jenkins)
* **CodeNarc CLI (as detailed above)**: Direct AST rule checking across script files.
* **`npm-groovy-lint` via `npx`**: CodeNarc bundled in a Node CLI runner:
  ```bash
  npx --yes npm-groovy-lint --path "buildenv/jenkins/" --rulesets "basic,concurrency" --noserver --output txt
  ```

### Option 2: Pipeline Unit Testing with `JenkinsPipelineUnit`
Simulate Jenkins CPS execution and step execution (`node`, `sh`, `stage`, `params`, `env`) locally using JUnit:
1. Create a temporary test harness directory: `/tmp/jenkins-unit-test`.
2. Add a `build.gradle` containing `com.lesfurets:JenkinsPipelineUnit:1.15` and `junit:junit:4.13.2`.
3. Create test cases extending `BasePipelineTest` that mock `params` and `env` and execute `runScript('Jenkinsfile')`.
4. Run `gradle -p /tmp/jenkins-unit-test test` to catch runtime binding and variable reference errors.

### Option 3: Testing Jenkins Job DSL Templates
For Job DSL scripts (like `testJobTemplate` or `perfL2JobTemplate`):
1. Download `job-dsl-core-standalone.jar`:
   ```bash
   curl -sLO https://repo.jenkins-ci.org/releases/org/jenkins-ci/plugins/job-dsl-core/1.87/job-dsl-core-1.87-standalone.jar
   ```
2. Run the DSL parser directly:
   ```bash
   java -jar job-dsl-core-1.87-standalone.jar /path/to/testJobTemplate
   ```

### Option 4: Local Jenkins Docker Instance
Execute the entire pipeline in an official Jenkins container:
```bash
docker run -d --name local-jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v "$(pwd):/repo:ro" \
  jenkins/jenkins:lts
```
Install the **Pipeline** and **Job DSL** plugins, then create a pipeline job pointing to `/repo`.

### Option 5: Local Test Suite Execution with TestKitGen (AQA TKG)
To run the underlying test suites (OpenJDK, functional, system) without Jenkins:
```bash
./get.sh -t /path/to/test-material
cd getDependency/testKitGen
make -f runtest.mk _jdk_math
```
