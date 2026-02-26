---
title: CI/CD Integration for CANoe Test Automation
impact: HIGH
impactDescription: Automated CANoe test execution in CI pipelines catches regressions early and enforces quality gates
tags: capl, external, ci-cd, jenkins, gitlab, automation, headless
---

## CI/CD Integration for CANoe Test Automation

Run CANoe tests headlessly in CI pipelines, parse results into standard formats, and manage licenses and artifacts for reliable automated testing.

**Headless execution from command line:**

```bash
"C:\Program Files\Vector CANoe 17\Exec64\CANoe64.exe" -d "config.cfg" \
    -t "TestModule1" \
    -r "reports\result.xml"
```

Key flags: `-d` for desktop/headless mode, `-t` to select test module, `-r` for report output path. CANoe exits with code 0 on pass, non-zero on failure.

**Jenkins pipeline (Jenkinsfile):**

```yaml
pipeline {
    agent { label 'canoe-runner' }
    stages {
        stage('Run CANoe Tests') {
            steps {
                bat '''
                    "C:\\Program Files\\Vector CANoe 17\\Exec64\\CANoe64.exe" ^
                        -d "configs\\integration_test.cfg" ^
                        -r "reports\\canoe_results.xml"
                '''
            }
        }
        stage('Publish Results') {
            steps {
                junit 'reports/*.xml'
                archiveArtifacts artifacts: 'reports/**', fingerprint: true
            }
        }
    }
    post {
        always {
            bat 'taskkill /F /IM CANoe64.exe /T 2>nul || exit /b 0'
        }
    }
}
```

Always force-kill CANoe in `post.always` — crashed runs leave orphan processes that block licenses.

**GitLab CI (.gitlab-ci.yml):**

```yaml
canoe_tests:
  stage: test
  tags:
    - windows
    - canoe
  script:
    - >
      & "C:\Program Files\Vector CANoe 17\Exec64\CANoe64.exe"
      -d "configs\integration_test.cfg"
      -r "reports\canoe_results.xml"
  after_script:
    - taskkill /F /IM CANoe64.exe /T 2>nul; exit 0
  artifacts:
    when: always
    paths:
      - reports/
    reports:
      junit: reports/*.xml
    expire_in: 30 days
```

**Converting CANoe reports to JUnit XML:**

CANoe's native XML report may not match JUnit schema. Use a conversion script:

```python
import xml.etree.ElementTree as ET

def canoe_to_junit(canoe_xml, output_path):
    tree = ET.parse(canoe_xml)
    root = tree.getroot()

    suite = ET.Element("testsuite", name="CANoe", tests="0", failures="0")
    pass_count = fail_count = 0

    for tc in root.iter("testcase"):
        name = tc.get("title", tc.get("name", "unknown"))
        verdict = tc.get("verdict", "none").lower()
        case = ET.SubElement(suite, "testcase", name=name, classname="CANoe")

        if verdict == "fail":
            ET.SubElement(case, "failure", message=f"Verdict: {verdict}")
            fail_count += 1
        else:
            pass_count += 1

    suite.set("tests", str(pass_count + fail_count))
    suite.set("failures", str(fail_count))
    ET.ElementTree(suite).write(output_path, xml_declaration=True)
```

**License management in CI:**

```bash
# Check license availability before running
"C:\Program Files\Vector\Licensing\VectorLicenseClient.exe" /check CANoe

# Use environment variable for license server
set VECTOR_LICENSE_SERVER=license-server.corp:8080
```

Dedicate a license pool for CI runners. If using floating licenses, add retry logic — a runner waiting for a license should not fail the pipeline immediately. Consider staggering pipeline schedules to avoid license contention.

**Artifact collection checklist:**

Collect these artifacts on every run (pass or fail):
- Test report XML (`reports/*.xml`)
- CANoe Write window log (`logs/*.txt`)
- Bus trace recordings (`traces/*.blf`)
- Screenshots on failure (if panel-based tests)

Use `when: always` (GitLab) or `post { always {} }` (Jenkins) to ensure artifacts are captured even on test failure.

Reference: Vector CANoe Help — Command Line Interface; Jenkins/GitLab CI documentation
