# Complete Guide: Adding a Custom Score Computation Method

This guide provides detailed step-by-step instructions to set up the CISO Assistant environment, add a new score computation method, and test it locally using Docker containers.

## Overview

You'll be:
1. ✅ Cloning and setting up the repository
2. ✅ Running the application with Docker
3. ✅ Creating a new scoring method (e.g., "MEDIAN" or "WEIGHTED_MEDIAN")
4. ✅ Writing comprehensive tests
5. ✅ Running and validating the tests
6. ✅ Verifying the changes work end-to-end

---

## Part 1: Initial Setup

### Prerequisites
- Docker >= 27.0
- Docker Compose
- Git
- 20GB free disk space
- For Windows: Docker Desktop with WSL2

### Step 1.1: Clone the Repository

```bash
# Navigate to your desired directory
cd ~/projects

# Clone the repository (uses the main branch with prebuilt images)
git clone --single-branch -b main https://github.com/intuitem/ciso-assistant-community.git

# Navigate into the directory
cd ciso-assistant-community

# Verify the structure
ls -la
# You should see: backend/, frontend/, docker-compose.yml, etc.
```

### Step 1.2: Verify Docker Installation

```bash
# Check Docker version
docker --version
# Expected: Docker version 27.0 or higher

# Check Docker Compose version
docker compose version
# Expected: Docker Compose version 2.0 or higher

# Test Docker access
docker run hello-world
```

---

## Part 2: Running the Application with Docker

### Step 2.1: Start the Containers

**On Linux/macOS:**
```bash
# Make the script executable
chmod +x docker-compose.sh

# Run the startup script
./docker-compose.sh
```

**On Windows (PowerShell):**
```powershell
# Run the startup script
.\docker-compose.ps1
```

### Step 2.2: Wait for Services to Be Ready

The script will display messages. Wait for all services to be healthy:

```
backend is healthy
huey is healthy
frontend is healthy
```

This usually takes 2-3 minutes.

### Step 2.3: Access the Application

Once ready, open your browser:
- **URL:** https://localhost:8443
- **Accept the self-signed certificate warning**

You'll be prompted to create a superuser:
```
Email: admin@example.com
Password: YourSecurePassword123
```

### Step 2.4: Verify the Application Works

1. Log in with your credentials
2. Navigate to a compliance assessment
3. Verify the dashboard loads without errors

---

## Part 3: Understanding the Current Code Structure

### Step 3.1: Locate Key Files

The scoring logic is in two main files:

**Backend Models:** `backend/core/models.py`
- Contains the `ComplianceAssessment` model
- Contains the `CalculationMethod` enum (AVG, SUM, AVG_OF_AVG)
- Contains `get_global_score()` method

**Tests:** `backend/core/tests/test_compliance_assessment_scoring.py`
- Comprehensive test suite for scoring methods
- Fixtures for test data setup

### Step 3.2: Current Scoring Methods

The `ComplianceAssessment.CalculationMethod` enum currently supports:

```python
class CalculationMethod(models.TextChoices):
    AVG = "avg", "Average"                          # Weighted average
    SUM = "sum", "Sum"                              # Weighted sum (raw)
    AVG_OF_AVG = "avg_of_avg", "Average of Averages"  # Hierarchical average
```

Each method:
- Excludes `NOT_APPLICABLE` requirements
- Excludes requirements with `is_scored=False`
- Returns `-1` if no scorable requirements exist
- Supports documentation scoring independently

---

## Part 4: Creating a Custom Scoring Method (MEDIAN Example)

### Step 4.1: Stop the Containers (Optional)

If you want to make code changes without the running application interfering:

```bash
# Stop containers
docker compose down

# Verify they're stopped
docker compose ps
```

### Step 4.2: Locate the ComplianceAssessment Model

```bash
# Open the models file
code backend/core/models.py  # VS Code
# Or: vim backend/core/models.py
```

Search for: `class ComplianceAssessment(`

Find the section with `CalculationMethod`:
```python
class CalculationMethod(models.TextChoices):
    AVG = "avg", "Average"
    SUM = "sum", "Sum"
    AVG_OF_AVG = "avg_of_avg", "Average of Averages"
```

### Step 4.3: Add Your New Calculation Method

Find the `CalculationMethod` class and add your method:

```python
class CalculationMethod(models.TextChoices):
    AVG = "avg", "Average"
    SUM = "sum", "Sum"
    AVG_OF_AVG = "avg_of_avg", "Average of Averages"
    MEDIAN = "median", "Median"  # NEW METHOD
```

### Step 4.4: Find the `get_global_score()` Method

Search for `def get_global_score(self)` in the `ComplianceAssessment` class.

This method contains logic like:
```python
def get_global_score(self):
    # ... existing code ...
    
    if self.score_calculation_method == ComplianceAssessment.CalculationMethod.AVG:
        impl_score = self._compute_avg_score(scored_ras, score_type)
    elif self.score_calculation_method == ComplianceAssessment.CalculationMethod.SUM:
        impl_score = self._compute_sum_score(scored_ras, score_type)
    elif self.score_calculation_method == ComplianceAssessment.CalculationMethod.AVG_OF_AVG:
        impl_score = self._compute_avg_of_avg_score(scored_ras)
    
    # ADD THIS:
    elif self.score_calculation_method == ComplianceAssessment.CalculationMethod.MEDIAN:
        impl_score = self._compute_median_score(scored_ras, score_type)
```

### Step 4.5: Implement the Helper Method

Add the implementation method in the `ComplianceAssessment` class:

```python
def _compute_median_score(self, scored_ras, score_type="score"):
    """
    Compute the median score.
    
    For requirements with scores: finds the middle value.
    Handles even/odd number of requirements.
    
    Args:
        scored_ras: QuerySet of RequirementAssessments to include
        score_type: "score" for implementation, "documentation_score" for docs
        
    Returns:
        Median value (float), or -1 if no requirements
    """
    import statistics
    
    if not scored_ras:
        return -1
    
    scores = []
    for ra in scored_ras:
        score_value = getattr(ra, score_type)
        if score_value is not None:
            scores.append(score_value)
    
    if not scores:
        return -1
    
    return statistics.median(scores)
```

### Step 4.6: Handle Max Score Calculation

Find the `get_total_max_score()` method and add your case:

```python
def get_total_max_score(self):
    # ... existing code for AVG, SUM, AVG_OF_AVG ...
    
    elif self.score_calculation_method == ComplianceAssessment.CalculationMethod.MEDIAN:
        return self.max_score  # Median is always between min and max
```

---

## Part 5: Creating Tests for Your New Method

### Step 5.1: Open the Test File

```bash
code backend/core/tests/test_compliance_assessment_scoring.py
```

### Step 5.2: Add Your Test Class

Add this test class at the end of the file (before any other test classes):

```python
@pytest.mark.django_db
class TestMedianScoringMethod:
    """Tests for the new MEDIAN scoring method."""

    def test_median_returns_middle_value(self, scoring_setup):
        """
        MEDIAN: finds the middle value.
        
        Scores: A1=80, A2=60, B1=40, B2=100
        Sorted: 40, 60, 80, 100
        Median = (60 + 80) / 2 = 70.0
        """
        ca = scoring_setup["ca"]
        ca.score_calculation_method = ComplianceAssessment.CalculationMethod.MEDIAN
        ca.save()

        scores = ca.get_global_score()
        assert scores["implementation_score"] == 70.0

    def test_median_with_odd_number_of_requirements(self, scoring_setup):
        """
        With odd number of requirements, median is the middle value.
        
        Scores: A1=80, A2=60, B1=40
        Sorted: 40, 60, 80
        Median = 60
        """
        ca = scoring_setup["ca"]
        
        # Mark B2 as not scored to get only 3 requirements
        ra_b2 = scoring_setup["ra_b2"]
        ra_b2.is_scored = False
        ra_b2.save()

        ca.score_calculation_method = ComplianceAssessment.CalculationMethod.MEDIAN
        ca.save()

        scores = ca.get_global_score()
        assert scores["implementation_score"] == 60

    def test_median_not_applicable_excluded(self, scoring_setup):
        """
        Requirements with NOT_APPLICABLE result should be excluded from median.
        
        Scores after exclusion: A1=80, A2=60, B1=40
        Sorted: 40, 60, 80
        Median = 60
        """
        ca = scoring_setup["ca"]
        ra_b2 = scoring_setup["ra_b2"]

        ra_b2.result = RequirementAssessment.Result.NOT_APPLICABLE
        ra_b2.save()

        ca.score_calculation_method = ComplianceAssessment.CalculationMethod.MEDIAN
        ca.save()

        scores = ca.get_global_score()
        assert scores["implementation_score"] == 60

    def test_median_unscored_excluded(self, scoring_setup):
        """
        Requirements with is_scored=False should be excluded from median.
        
        Scores after exclusion: A1=80, A2=60, B1=40
        Sorted: 40, 60, 80
        Median = 60
        """
        ca = scoring_setup["ca"]
        ra_b2 = scoring_setup["ra_b2"]

        RequirementAssessment.objects.filter(pk=ra_b2.pk).update(is_scored=False)

        ca.score_calculation_method = ComplianceAssessment.CalculationMethod.MEDIAN
        ca.save()

        scores = ca.get_global_score()
        assert scores["implementation_score"] == 60

    def test_median_returns_minus_one_when_no_scored_requirements(self, scoring_setup):
        """When there are no scored requirements, median should return -1."""
        ca = scoring_setup["ca"]

        RequirementAssessment.objects.filter(compliance_assessment=ca).update(
            is_scored=False
        )

        ca.score_calculation_method = ComplianceAssessment.CalculationMethod.MEDIAN
        ca.save()

        scores = ca.get_global_score()
        assert scores["implementation_score"] == -1

    def test_median_max_score(self, scoring_setup):
        """Median max score should always be the framework max_score."""
        ca = scoring_setup["ca"]
        ca.score_calculation_method = ComplianceAssessment.CalculationMethod.MEDIAN
        ca.save()

        assert ca.get_total_max_score() == 100

    def test_median_with_documentation_scoring(self, scoring_setup):
        """
        Test median with documentation scoring enabled.
        
        Implementation scores: 80, 60, 40, 100 → median = 70
        Documentation scores: 90, 70, 50, 80 → median = 75
        Maturity: (70 + 75) / 2 = 72.5
        """
        ca = scoring_setup["ca"]
        ca.score_calculation_method = ComplianceAssessment.CalculationMethod.MEDIAN
        ca.show_documentation_score = True
        ca.save()

        # Set documentation scores
        scoring_setup["ra_a1"].documentation_score = 90
        scoring_setup["ra_a1"].save()
        scoring_setup["ra_a2"].documentation_score = 70
        scoring_setup["ra_a2"].save()
        scoring_setup["ra_b1"].documentation_score = 50
        scoring_setup["ra_b1"].save()
        scoring_setup["ra_b2"].documentation_score = 80
        scoring_setup["ra_b2"].save()

        scores = ca.get_global_score()
        assert scores["implementation_score"] == 70.0
        assert scores["documentation_score"] == 75.0
        assert scores["maturity_score"] == 72.5

    def test_median_single_requirement(self, scoring_setup):
        """With only one scored requirement, median equals that score."""
        ca = scoring_setup["ca"]
        
        # Mark all but one as not scored
        RequirementAssessment.objects.filter(compliance_assessment=ca).exclude(
            pk=scoring_setup["ra_a1"].pk
        ).update(is_scored=False)

        ca.score_calculation_method = ComplianceAssessment.CalculationMethod.MEDIAN
        ca.save()

        scores = ca.get_global_score()
        assert scores["implementation_score"] == 80
```

### Step 5.3: Verify Test File Syntax

```bash
cd backend

# Run just a syntax check
python -m py_compile core/tests/test_compliance_assessment_scoring.py

# Expected: no output (success)
```

---

## Part 6: Running the Tests

### Step 6.1: Prepare the Backend Environment

```bash
# Navigate to backend directory
cd backend

# Install test dependencies (if using local Python, not Docker)
# uv sync

# Or, if using Docker, ensure containers are running:
# From the root directory:
cd ..
docker compose up -d
```

### Step 6.2: Run the Tests Inside Docker

**Option A: Run only your new tests**

```bash
# Execute pytest in the running backend container
docker compose exec backend pytest core/tests/test_compliance_assessment_scoring.py::TestMedianScoringMethod -v

# Expected output:
# core/tests/test_compliance_assessment_scoring.py::TestMedianScoringMethod::test_median_returns_middle_value PASSED
# core/tests/test_compliance_assessment_scoring.py::TestMedianScoringMethod::test_median_with_odd_number_of_requirements PASSED
# ... etc
```

**Option B: Run all compliance assessment scoring tests**

```bash
docker compose exec backend pytest core/tests/test_compliance_assessment_scoring.py -v

# This runs all scoring tests (AVG, SUM, AVG_OF_AVG, and your MEDIAN)
```

**Option C: Run with coverage report**

```bash
docker compose exec backend pytest core/tests/test_compliance_assessment_scoring.py::TestMedianScoringMethod -v --cov=core.models --cov-report=term-missing
```

**Option D: Run with detailed output and debugging**

```bash
docker compose exec backend pytest core/tests/test_compliance_assessment_scoring.py::TestMedianScoringMethod -vv -s

# -vv: extra verbose (shows test details)
# -s: show print statements (if you added debugging)
```

### Step 6.3: Expected Test Output

Successful output should look like:

```
======= test session starts =======
platform linux -- Python 3.14.0, pytest-4.11.1, py-1.11.0, pluggy-1.0.0
django: settings: ciso_assistant.settings
rootdir: /app/backend, configfile: pytest.ini
plugins: django-4.11.1, html-4.1.1
collected 8 items

core/tests/test_compliance_assessment_scoring.py::TestMedianScoringMethod::test_median_returns_middle_value PASSED [ 12%]
core/tests/test_compliance_assessment_scoring.py::TestMedianScoringMethod::test_median_with_odd_number_of_requirements PASSED [ 25%]
core/tests/test_compliance_assessment_scoring.py::TestMedianScoringMethod::test_median_not_applicable_excluded PASSED [ 37%]
core/tests/test_compliance_assessment_scoring.py::TestMedianScoringMethod::test_median_unscored_excluded PASSED [ 50%]
core/tests/test_compliance_assessment_scoring.py::TestMedianScoringMethod::test_median_returns_minus_one_when_no_scored_requirements PASSED [ 62%]
core/tests/test_compliance_assessment_scoring.py::TestMedianScoringMethod::test_median_max_score PASSED [ 75%]
core/tests/test_compliance_assessment_scoring.py::TestMedianScoringMethod::test_median_with_documentation_scoring PASSED [ 87%]
core/tests/test_compliance_assessment_scoring.py::TestMedianScoringMethod::test_median_single_requirement PASSED [100%]

======= 8 passed in 2.34s =======
```

### Step 6.4: If Tests Fail

**Debug approach:**

```bash
# Run a single test with verbose output
docker compose exec backend pytest core/tests/test_compliance_assessment_scoring.py::TestMedianScoringMethod::test_median_returns_middle_value -vv -s

# -vv: extra verbose
# -s: show print statements
```

**Common issues and solutions:**

| Issue | Solution |
|-------|----------|
| `AttributeError: module has no attribute 'CalculationMethod.MEDIAN'` | Verify the method was added to the enum correctly |
| `TypeError: _compute_median_score() missing 1 required positional argument` | Check method signature and indentation in the class |
| `ImportError: cannot import name 'statistics'` | Add `import statistics` at the top of models.py |
| `AssertionError: 70.0 != 75.5` | Verify the test data matches the expected calculation |

**Add debugging to your test:**

```python
def test_median_returns_middle_value(self, scoring_setup):
    ca = scoring_setup["ca"]
    ca.score_calculation_method = ComplianceAssessment.CalculationMethod.MEDIAN
    ca.save()

    scores = ca.get_global_score()
    
    # DEBUG: Print actual values
    print(f"Implementation score: {scores['implementation_score']}")
    print(f"Documentation score: {scores['documentation_score']}")
    print(f"Maturity score: {scores['maturity_score']}")
    
    assert scores["implementation_score"] == 70.0
```

Then run with `-s` flag to see the debug output:
```bash
docker compose exec backend pytest core/tests/test_compliance_assessment_scoring.py::TestMedianScoringMethod::test_median_returns_middle_value -vv -s
```

---

## Part 7: Updating the Serializer (Optional)

The API needs to know about your new method. Update the serializer:

### Step 7.1: Find the Serializer

```bash
# Open serializers file
code backend/core/serializers.py
```

Search for: `score_calculation_method`

Find the field definition (usually looks like):
```python
score_calculation_method = serializers.ChoiceField(
    choices=ComplianceAssessment.CalculationMethod.choices
)
```

**No changes needed!** The `TextChoices` enum automatically includes your new method in the choices.

---

## Part 8: Making Changes to Running Containers

If the containers are running and you've made code changes:

### Step 8.1: Rebuild the Backend Container

```bash
# Option 1: Rebuild from scratch (recommended for code changes)
docker compose down
docker compose build --no-cache backend
docker compose up -d

# Option 2: Just restart the backend
docker compose restart backend
```

### Step 8.2: Verify the Container Started

```bash
# Wait 10-15 seconds, then check health
docker compose ps

# You should see:
# backend     ... healthy
# frontend    ... healthy
```

### Step 8.3: Clear Django Cache (if needed)

```bash
# For Django cache
docker compose exec backend python manage.py clear_cache

# Or restart both backend and huey
docker compose restart backend huey
```

---

## Part 9: End-to-End Testing in the UI

### Step 9.1: Access the Application

```
https://localhost:8443
```

Login with your admin credentials.

### Step 9.2: Create a Test Compliance Assessment

1. Go to **Governance → Compliance Assessments**
2. Click **"+ New Assessment"**
3. Select a framework (e.g., **ISO 27001**)
4. Click **Create**
5. Add a name and description

### Step 9.3: Add Scores to Requirements

1. In your new assessment, go to **Requirement Assessments**
2. Select multiple requirements
3. Add implementation scores (0-100) to each
4. Save changes

### Step 9.4: Change the Scoring Method

1. Go back to the main assessment view
2. Click **Settings** or **Configuration** (usually at the top of the card)
3. Find **Score Calculation Method** dropdown
4. Select **"Median"** (your new method)
5. Save changes

### Step 9.5: Verify the Score Updates

1. The overall compliance score card should update immediately
2. The score should display the median value of all requirement scores
3. Example:
   - Requirement scores: 40, 60, 80, 100
   - Median: 70
   - Should display in the compliance score widget

### Step 9.6: Test Documentation Scoring (Optional)

1. In the assessment settings, enable **"Show Documentation Scores"**
2. Go back and add documentation scores to requirements
3. Verify the score updates:
   - Implementation score = median of implementation scores
   - Documentation score = median of documentation scores
   - Maturity score = average of implementation and documentation

---

## Part 10: Running All Tests

### Step 10.1: Run Full Compliance Scoring Test Suite

```bash
# All compliance assessment scoring tests
docker compose exec backend pytest core/tests/test_compliance_assessment_scoring.py -v

# Count: should see ~40+ tests passing
```

### Step 10.2: Run All Backend Tests

```bash
# All backend tests (takes 5-10 minutes)
docker compose exec backend pytest core/tests/ -v

# Or with coverage
docker compose exec backend pytest core/tests/ --cov=core.models --cov-report=html
```

### Step 10.3: Generate Coverage Report

```bash
# Generate HTML coverage report
docker compose exec backend pytest core/tests/test_compliance_assessment_scoring.py --cov=core.models --cov-report=html

# Copy report out of container
docker compose cp backend:/app/backend/htmlcov ./coverage

# Open in browser: coverage/index.html
```

### Step 10.4: Check for Performance Regressions

```bash
# Run performance-focused tests
docker compose exec backend pytest core/tests/test_scoring.py::TestGlobalScoreQueryPerformance -v

# Verifies get_global_score() doesn't cause N+1 database queries
```

---

## Part 11: Deploying Your Changes (Production)

### Step 11.1: Create a Migration (if you added database fields)

```bash
# Since we only added a choice to TextChoices, no migration needed
# But if you added fields, run:
docker compose exec backend python manage.py makemigrations core
docker compose exec backend python manage.py migrate
```

### Step 11.2: Commit Your Changes

```bash
git add backend/core/models.py
git add backend/core/tests/test_compliance_assessment_scoring.py
git commit -m "feat: add MEDIAN scoring method for compliance assessments"
git push
```

### Step 11.3: Build Docker Images (if needed)

For production deployment with your changes:

```bash
# Linux/macOS
./docker-compose-build.sh

# Windows
.\docker-compose-build.ps1
```

### Step 11.4: Create a Pull Request

1. Push your branch to GitHub
2. Create a PR with description of the new scoring method
3. Include test results showing all tests pass
4. Link any related issues

---

## Troubleshooting

### Issue: Tests don't run / pytest not found

```bash
# Make sure pytest is installed in the container
docker compose exec backend pip list | grep pytest

# If not installed, restart the container (it should install dependencies)
docker compose restart backend

# Wait 30 seconds for startup
sleep 30
docker compose ps
```

### Issue: Container out of memory during tests

```bash
# Stop and increase Docker memory
docker compose down

# Linux/Mac: increase in Docker settings (Preferences > Resources)
# Windows: increase in Docker Desktop settings

# Restart with increased memory
docker compose up -d
```

### Issue: Port 8443 already in use

```bash
# Find what's using port 8443
lsof -i :8443              # macOS/Linux
netstat -ano | findstr :8443  # Windows

# Kill the process or use a different port
# Edit docker-compose.yml:
#   ports:
#     - "9443:8443"  # Use port 9443 instead
```

### Issue: Changes not reflected in running container

```bash
# Full rebuild (nuclear option)
docker compose down --volumes  # WARNING: deletes database!
docker compose build --no-cache
docker compose up -d

# Or safer - just rebuild backend
docker compose down
docker compose build --no-cache backend
docker compose up -d
```

### Issue: Database migration errors

```bash
# Check migration status
docker compose exec backend python manage.py showmigrations

# Apply pending migrations
docker compose exec backend python manage.py migrate

# Reset database (WARNING: deletes data!)
docker compose exec backend python manage.py flush --no-input
```

### Issue: Can't connect to localhost:8443

```bash
# Check if containers are running
docker compose ps

# Check container logs
docker compose logs frontend
docker compose logs backend

# Try accessing with IP instead of localhost
# Find backend IP:
docker inspect ciso-assistant-community-backend-1 | grep IPAddress
```

---

## Quick Reference Commands

```bash
# ========== CONTAINER MANAGEMENT ==========
./docker-compose.sh              # Start all containers (Linux/macOS)
.\docker-compose.ps1             # Start all containers (Windows)
docker compose down              # Stop all containers
docker compose restart backend   # Restart just backend
docker compose ps                # Show container status

# ========== LOGS ==========
docker compose logs -f backend   # Backend logs (follow)
docker compose logs -f frontend  # Frontend logs (follow)
docker compose logs -f huey      # Task queue logs (follow)
docker compose logs backend | tail -50  # Last 50 lines

# ========== TESTING ==========
docker compose exec backend pytest core/tests/test_compliance_assessment_scoring.py::TestMedianScoringMethod -v
docker compose exec backend pytest core/tests/test_compliance_assessment_scoring.py -v
docker compose exec backend pytest core/tests/ -v --cov=core.models

# ========== DATABASE ==========
docker compose exec backend python manage.py dbshell
docker compose exec backend python manage.py showmigrations
docker compose exec backend python manage.py migrate
docker compose exec backend python manage.py flush --no-input

# ========== USER MANAGEMENT ==========
docker compose exec backend python manage.py createsuperuser
docker compose exec backend python manage.py changepassword admin

# ========== DEBUGGING ==========
docker compose exec backend python manage.py shell
docker compose exec backend python -m pdb manage.py test
docker compose exec backend python manage.py runserver 0.0.0.0:8000

# ========== CLEANUP ==========
docker system prune -a           # Remove unused containers/images
docker compose down -v           # Remove containers AND volumes (delete DB!)
docker volume ls                 # List all volumes
```

---

## Advanced: Custom Scoring Method Examples

### Example 1: Weighted Median

```python
def _compute_weighted_median_score(self, scored_ras, score_type="score"):
    """
    Compute weighted median (weighted percentile 50).
    
    Accounts for requirement weights in the median calculation.
    """
    if not scored_ras:
        return -1
    
    scores_with_weights = []
    for ra in scored_ras:
        score_value = getattr(ra, score_type)
        if score_value is not None:
            weight = ra.requirement.weight or 1
            scores_with_weights.append((score_value, weight))
    
    if not scores_with_weights:
        return -1
    
    # Sort by score
    scores_with_weights.sort(key=lambda x: x[0])
    
    # Calculate cumulative weight
    total_weight = sum(w for _, w in scores_with_weights)
    cumulative_weight = 0
    target_weight = total_weight / 2
    
    for score, weight in scores_with_weights:
        cumulative_weight += weight
        if cumulative_weight >= target_weight:
            return score
    
    return scores_with_weights[-1][0]
```

### Example 2: Percentile-Based

```python
def _compute_percentile_score(self, scored_ras, percentile=50, score_type="score"):
    """
    Compute a specific percentile score (e.g., 25th, 50th, 75th).
    """
    import numpy as np
    
    if not scored_ras:
        return -1
    
    scores = []
    for ra in scored_ras:
        score_value = getattr(ra, score_type)
        if score_value is not None:
            scores.append(score_value)
    
    if not scores:
        return -1
    
    return np.percentile(scores, percentile)
```

### Example 3: Risk-Weighted Average

```python
def _compute_risk_weighted_score(self, scored_ras, score_type="score"):
    """
    Score weighted by the risk level of each requirement.
    Higher-risk requirements get more weight in the average.
    """
    if not scored_ras:
        return -1
    
    total_score = 0
    total_weight = 0
    
    for ra in scored_ras:
        score_value = getattr(ra, score_type)
        if score_value is not None:
            # Get risk level from requirement (if available)
            risk_multiplier = getattr(ra.requirement, 'risk_level', 1)
            weight = (ra.requirement.weight or 1) * risk_multiplier
            
            total_score += score_value * weight
            total_weight += weight
    
    if total_weight == 0:
        return -1
    
    return total_score / total_weight
```

---

## Testing Your Custom Implementation

When creating custom scoring methods, test these scenarios:

```python
@pytest.mark.django_db
class TestYourCustomMethod:
    """Template for testing custom scoring methods."""

    def test_basic_calculation(self, scoring_setup):
        """Test the basic calculation matches expected behavior."""
        ca = scoring_setup["ca"]
        ca.score_calculation_method = ComplianceAssessment.CalculationMethod.YOUR_METHOD
        ca.save()
        scores = ca.get_global_score()
        # Assert expected value
        assert scores["implementation_score"] == expected_value

    def test_excludes_not_applicable(self, scoring_setup):
        """Test that NOT_APPLICABLE requirements are excluded."""
        ca = scoring_setup["ca"]
        scoring_setup["ra_b2"].result = RequirementAssessment.Result.NOT_APPLICABLE
        scoring_setup["ra_b2"].save()
        ca.score_calculation_method = ComplianceAssessment.CalculationMethod.YOUR_METHOD
        ca.save()
        scores = ca.get_global_score()
        # Assert NA requirement excluded

    def test_excludes_unscored(self, scoring_setup):
        """Test that is_scored=False requirements are excluded."""
        ca = scoring_setup["ca"]
        RequirementAssessment.objects.filter(pk=scoring_setup["ra_b2"].pk).update(is_scored=False)
        ca.score_calculation_method = ComplianceAssessment.CalculationMethod.YOUR_METHOD
        ca.save()
        scores = ca.get_global_score()
        # Assert unscored requirement excluded

    def test_returns_minus_one_when_empty(self, scoring_setup):
        """Test that -1 is returned when no scored requirements."""
        ca = scoring_setup["ca"]
        RequirementAssessment.objects.filter(compliance_assessment=ca).update(is_scored=False)
        ca.score_calculation_method = ComplianceAssessment.CalculationMethod.YOUR_METHOD
        ca.save()
        scores = ca.get_global_score()
        assert scores["implementation_score"] == -1

    def test_with_documentation_scoring(self, scoring_setup):
        """Test with documentation scoring enabled."""
        ca = scoring_setup["ca"]
        ca.score_calculation_method = ComplianceAssessment.CalculationMethod.YOUR_METHOD
        ca.show_documentation_score = True
        ca.save()
        # Set doc scores...
        scores = ca.get_global_score()
        # Assert three scores computed correctly

    def test_max_score_calculation(self, scoring_setup):
        """Test that max score is calculated correctly."""
        ca = scoring_setup["ca"]
        ca.score_calculation_method = ComplianceAssessment.CalculationMethod.YOUR_METHOD
        ca.save()
        max_score = ca.get_total_max_score()
        # Assert max score is correct
```

---

## Next Steps

1. **Customize your scoring method:** Modify the calculation logic based on your needs
2. **Add more tests:** Cover edge cases specific to your use case
3. **Update documentation:** Add user-facing docs explaining the new method
4. **Submit a PR:** Share your improvement with the CISO Assistant community
5. **Monitor performance:** Use profiling tools to ensure efficient queries

---

## Support & Resources

- **GitHub Issues:** https://github.com/intuitem/ciso-assistant-community/issues
- **Discord Community:** https://discord.gg/qvkaMdQ8da
- **Official Docs:** https://intuitem.gitbook.io/ciso-assistant
- **Development Guide:** Check `CONTRIBUTING.md` in the repository

---

## Common Pitfalls

| Pitfall | Prevention |
|---------|-----------|
| Forgetting to handle None values | Always check `if value is not None:` |
| N+1 database queries | Use `select_related()` and `prefetch_related()` |
| Calculations not matching tests | Print intermediate values during debugging |
| Container using old code | Run `docker compose build --no-cache` |
| Tests passing but UI broken | Test the API endpoint directly with curl/Postman |
| Database locks during testing | Don't run tests in parallel on shared DB |

---

**Happy coding! 🚀**
