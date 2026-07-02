# PROMPT: Build “Rekalibrasi ISO 5167 Lab” — Interactive Orifice Flow Calculator & Simulator for GitHub Pages

You are a senior instrumentation engineer, process measurement specialist, and frontend software architect. Build a fully interactive web application named:

**Rekalibrasi ISO 5167 Lab**

Subtitle: **Interactive Differential Pressure Flow Measurement & Orifice Plate Simulator**

The application must run fully as a static frontend application deployable to GitHub Pages. Do not require a backend, server, database, Docker, or authentication. All calculations, simulations, validation, charting, reporting, and saving presets must run in the browser.

Use:

- Vite
- React
- TypeScript
- Tailwind CSS
- shadcn/ui or clean reusable UI components
- Recharts or Plotly for charts
- Zustand or React state for global state
- localStorage or IndexedDB for saved presets
- React Router with HashRouter for GitHub Pages compatibility
- SVG-based interactive diagrams for pipe, orifice plate, taps, and flow conditioner layout

The application must be responsive for desktop, tablet, and mobile.

---

## 1. Product Goal

Create an interactive engineering learning and pre-sizing tool for differential pressure flow measurement based on ISO 5167 concepts.

The application should help users:

1. Understand the working principle of DP flow measurement.
2. Calculate mass flowrate and volumetric flowrate through an orifice plate.
3. Solve forward and reverse orifice calculation cases.
4. Simulate tapping arrangements.
5. Check installation quality.
6. Explore sensitivity of flow to differential pressure, beta ratio, density, viscosity, and tapping selection.
7. Estimate uncertainty.
8. Export calculation reports.
9. Save and reload local calculation scenarios.

The app is an engineering estimator and educational simulator, not a certified custody-transfer or safety-critical sizing tool.

Add this disclaimer clearly in the app:

> This application is for engineering learning, preliminary calculation, and sanity-checking only. It does not replace the official ISO 5167 standard, certified flow computer software, vendor sizing software, laboratory calibration, or review by a qualified engineer.

---

## 2. Important Standard Scope and Limitation

Implement these rules in the UI and validation engine:

- ISO 5167 applies to differential pressure devices inserted in circular conduits running full.
- Supported conceptual primary devices:
  - Orifice plate
  - Nozzle
  - Venturi tube
- For MVP, focus deeply on **orifice plate**.
- Fluid must be single-phase.
- Flow must be subsonic.
- Pulsating flow is not supported.
- The pipe must run full at the measurement section.
- For exact ISO-compliant orifice calculations, ISO 5167-2 equations, limits, and installation tables are required.
- If exact ISO 5167-2 coefficients/tables are not implemented, the app must label results as:
  - `EDUCATIONAL_ESTIMATE`
  - `PRELIMINARY`
  - or `NOT_FULL_ISO_5167_2_COMPLIANT`

Do not copy large copyrighted ISO text or tables into the app. Implement formulas, summaries, engineering warnings, and user-facing explanations in original wording.

---

## 3. Main Navigation

Create the following pages/modules:

```text
Dashboard
Orifice Calculator
Tapping Simulator
Installation Checker
Flow Simulation
Uncertainty Lab
Flow Conditioner Advisor
Fluid Properties
Report / Export
Reference Notes
Settings
```

Use a sidebar on desktop and bottom navigation or collapsible drawer on mobile.

---

## 4. Dashboard Design

The dashboard should show an industrial instrumentation cockpit.

Cards:

1. **Orifice ISO 5167 Calculator**
   - Calculate Q from ΔP, D, d, density, viscosity.
2. **Tapping Layout Simulator**
   - Visualize corner, flange, and D-D/2 tapping.
3. **Installation Checker**
   - Check straight run, fittings, swirl, pipe condition, and flow conditioner need.
4. **Flow Simulation**
   - Plot ΔP vs Q, β vs Q, Re vs C, uncertainty sensitivity.
5. **Uncertainty Lab**
   - Estimate combined uncertainty and contribution breakdown.
6. **Report Generator**
   - Export calculation summary as JSON, CSV, and printable HTML/PDF.

Top status indicators:

- Calculation Mode
- Fluid Mode
- Compliance Status
- Saved Scenario
- Theme Toggle

---

## 5. Core Data Model

Create strongly typed TypeScript models.

```ts
export type FluidType = "liquid" | "gas" | "steam" | "custom";

export type PrimaryDeviceType =
  | "orifice_plate"
  | "nozzle"
  | "venturi_tube";

export type OrificeTappingType =
  | "corner"
  | "flange"
  | "d_d2";

export type CalculationMode =
  | "solve_flow_from_dp"
  | "solve_dp_from_flow"
  | "solve_orifice_bore_from_flow"
  | "solve_beta_from_flow";

export type ComplianceStatus =
  | "VALID"
  | "VALID_WITH_WARNING"
  | "OUT_OF_RANGE"
  | "NOT_FULL_ISO_5167_2_COMPLIANT"
  | "EDUCATIONAL_ESTIMATE";

export interface FluidProperties {
  fluidType: FluidType;
  name: string;
  densityKgM3: number;
  dynamicViscosityPaS: number;
  temperatureC: number;
  pressureAbsPa: number;
  isentropicExponent?: number;
  compressibilityFactor?: number;
  vaporPressurePa?: number;
}

export interface PipeGeometry {
  pipeInsideDiameterM: number;
  pipeMaterial?: string;
  pipeRoughnessM?: number;
  pipeTemperatureC?: number;
  pipeThermalExpansionCoeff?: number;
}

export interface OrificeGeometry {
  orificeBoreM: number;
  beta: number;
  plateMaterial?: string;
  plateTemperatureC?: number;
  plateThermalExpansionCoeff?: number;
  edgeCondition?: "sharp_square" | "worn" | "unknown";
}

export interface OrificeCalculationInput {
  mode: CalculationMode;
  primaryDevice: "orifice_plate";
  tappingType: OrificeTappingType;
  fluid: FluidProperties;
  pipe: PipeGeometry;
  orifice: OrificeGeometry;
  differentialPressurePa?: number;
  massFlowKgS?: number;
  volumeFlowM3S?: number;
  dischargeCoefficientMode: "manual" | "educational_default" | "iso5167_2_plugin";
  manualDischargeCoefficient?: number;
  expansibilityMode: "incompressible" | "manual" | "iso5167_2_plugin";
  manualExpansibilityFactor?: number;
}

export interface OrificeCalculationResult {
  massFlowKgS: number;
  volumeFlowM3S: number;
  differentialPressurePa: number;
  beta: number;
  pipeReynoldsNumber: number;
  orificeReynoldsNumber: number;
  dischargeCoefficient: number;
  expansibilityFactor: number;
  velocityPipeMS: number;
  velocityOrificeMS: number;
  complianceStatus: ComplianceStatus;
  warnings: string[];
  assumptions: string[];
}
```

---

## 6. Core Formula Engine

Create files:

```text
src/lib/formulas/orificeIso5167.ts
src/lib/formulas/reynolds.ts
src/lib/formulas/uncertainty.ts
src/lib/formulas/unitConversion.ts
src/lib/formulas/iterativeSolver.ts
src/lib/formulas/validation.ts
```

### 6.1 Basic Orifice Equation

Implement the core differential pressure flow equation:

```ts
qm = C * epsilon * (Math.PI / 4) * d ** 2 * Math.sqrt(2 * deltaP * rho1) / Math.sqrt(1 - beta ** 4);
```

Where:

- `qm` = mass flowrate, kg/s
- `C` = discharge coefficient
- `epsilon` = expansibility factor
- `d` = orifice bore under working condition, m
- `beta = d / D`
- `deltaP` = differential pressure, Pa
- `rho1` = upstream density, kg/m³

Then:

```ts
qv = qm / rho;
```

For incompressible liquid:

```ts
epsilon = 1;
```

For gas/compressible mode:

- allow manual epsilon input in MVP
- prepare plugin placeholder for ISO 5167-2 expansibility equation
- warn if exact equation is not implemented

### 6.2 Reynolds Number

Implement:

```ts
ReD = (4 * qm) / (Math.PI * mu * D);
Red = ReD / beta;
```

Also compute:

```ts
Vpipe = qv / (Math.PI * D ** 2 / 4);
Vorifice = qv / (Math.PI * d ** 2 / 4);
```

### 6.3 Discharge Coefficient Handling

Support three modes:

1. Manual C
2. Educational default C
3. ISO 5167-2 plugin mode

For MVP:

- default manual C = 0.606
- clearly mark as educational default
- do not claim full ISO 5167-2 compliance unless exact equation and limits are implemented

Create interface:

```ts
export interface DischargeCoefficientModel {
  id: string;
  label: string;
  compute(input: OrificeCalculationInput, reynolds: number): {
    C: number;
    warnings: string[];
    complianceStatus: ComplianceStatus;
  };
}
```

Create placeholder:

```ts
const iso5167Part2ReaderHarrisGallagherModel: DischargeCoefficientModel = {
  id: "iso5167_2_plugin",
  label: "ISO 5167-2 Orifice Model Plugin",
  compute() {
    return {
      C: NaN,
      warnings: [
        "ISO 5167-2 exact discharge coefficient equation is not yet implemented. Upload or verify ISO 5167-2 source before enabling full compliance."
      ],
      complianceStatus: "NOT_FULL_ISO_5167_2_COMPLIANT"
    };
  }
};
```

### 6.4 Iterative Solver

Create generic secant/bisection hybrid solver.

Support solving:

- flow from DP
- DP from flow
- orifice bore from target flow
- beta from target flow

Requirements:

- max iterations configurable
- convergence tolerance configurable
- return iteration table
- display iteration history in UI
- handle invalid ranges gracefully

```ts
export interface IterationStep {
  iteration: number;
  x: number;
  y: number;
  error: number;
}

export interface SolverResult<T> {
  value: T;
  converged: boolean;
  iterations: IterationStep[];
  warnings: string[];
}
```

For reverse calculation, use root finding:

```ts
f(x) = calculatedFlow(x) - targetFlow;
```

Use bisection fallback when secant becomes unstable.

---

## 7. Orifice Calculator UI

Create page:

```text
src/pages/OrificeCalculatorPage.tsx
```

Layout:

- Left: input form
- Center: calculation result cards
- Right: formula, assumptions, compliance, warnings
- Bottom: chart and iteration table

### Inputs

Sections:

#### Calculation Mode

- Solve Q from ΔP
- Solve ΔP from Q
- Solve bore d from Q
- Solve beta from Q

#### Fluid

- Fluid type: liquid/gas/custom
- Density
- Dynamic viscosity
- Temperature
- Absolute pressure
- Isentropic exponent
- Compressibility factor
- Vapor pressure, optional

#### Pipe

- Pipe ID D
- Material
- Roughness
- Temperature
- Thermal expansion coefficient

#### Orifice

- Bore d
- Beta β
- Tapping type
- Edge condition
- Plate material
- Temperature correction toggle

#### Measurement

- Differential pressure
- Static pressure
- DP transmitter range
- DP noise/pulsation input, optional

### Outputs

Show:

- Mass flow kg/s
- Volumetric flow m³/h
- Pipe velocity m/s
- Orifice velocity m/s
- ReD
- Red
- C
- ε
- β
- ΔP
- Compliance status
- Warning badges

Status colors:

- green: valid
- yellow: warning
- red: out of range
- blue: educational estimate

---

## 8. Tapping Simulator

Create:

```text
src/pages/TappingSimulatorPage.tsx
src/components/flow/TappingLayoutSvg.tsx
```

Use SVG to display:

- horizontal pipe
- flow arrow
- orifice plate
- upstream pressure tap
- downstream pressure tap
- labels p1 and p2
- dimension markers
- D, d, β
- tap distance labels

Supported tapping types:

1. Corner taps
2. Flange taps
3. D and D/2 taps

When tapping type changes, animate tap position.

Add explanation panel:

- what the tapping type means
- typical conceptual location
- whether exact implementation depends on ISO 5167-2
- warning that vena contracta taps are not part of ISO 5167-1/2 scope from the provided source

Add slider:

- move upstream tap position
- move downstream tap position
- show compliance badge:
  - “Conceptual only”
  - “Matches selected tapping layout”
  - “Non-standard tap position”

Do not claim exact compliance unless ISO 5167-2 details are implemented.

---

## 9. Installation Checker

Create:

```text
src/pages/InstallationCheckerPage.tsx
src/lib/installation/installationRules.ts
```

Inputs:

- upstream fitting type:
  - long straight pipe
  - single 90 elbow
  - double elbow same plane
  - double elbow different plane
  - tee
  - reducer
  - expander
  - partially closed valve
  - pump discharge
  - unknown
- downstream fitting type
- available upstream straight length in D
- available downstream straight length in D
- flow conditioner installed?
- flow conditioner type
- pipe straightness condition
- pipe circularity condition
- pipe cleanliness
- pipe roughness known?
- drain/vent near primary device?
- temperature pocket location

Outputs:

- installation quality score
- warnings
- suggested action
- “needs flow conditioner” recommendation
- “additional uncertainty may apply” note

Important:

- Since exact minimum straight length tables for orifice plates are in ISO 5167-2, use a rule placeholder.
- Show “requires ISO 5167-2 table verification” when exact fitting table is needed.
- Provide educational estimates but do not hard-code undocumented values.

---

## 10. Flow Conditioner Advisor

Create:

```text
src/pages/FlowConditionerAdvisorPage.tsx
```

Support conceptual explanation and selection:

- tube bundle straightener
- AMCA straightener
- Étoile straightener
- Gallagher flow conditioner
- K-Lab NOVA perforated plate
- NEL/Spearman flow conditioner
- Sprenkle conditioner
- Zanker flow conditioner
- Zanker plate

For each:

- schematic SVG or simple geometric drawing
- purpose
- whether it reduces swirl only or redistributes profile
- approximate pressure loss coefficient K if available from provided source
- warning that compliance depends on test and applicable standard section

Add calculation:

```ts
deltaPc = K * rho * V ** 2 / 2;
```

Display:

- additional pressure loss across conditioner
- effect on system pressure budget
- “not a substitute for compliance test” warning

---

## 11. Flow Simulation Page

Create:

```text
src/pages/FlowSimulationPage.tsx
```

Interactive simulations:

### A. ΔP vs Q curve

- x-axis: ΔP
- y-axis: Q
- user sets D, d, ρ, μ, C, ε
- show square-root relationship

### B. β sensitivity

- x-axis: β
- y-axis: Q
- fixed D and ΔP
- show how higher beta increases flow but affects validity/warnings

### C. Density sensitivity

- x-axis: density
- y-axis: Q
- useful for liquid/gas property impact

### D. Viscosity/Reynolds effect

- x-axis: viscosity
- y-axis: ReD
- show warning at low Reynolds

### E. Tapping comparison

- compare conceptual tapping types
- do not change C unless exact ISO 5167-2 implementation exists
- show note: “Tapping selection affects coefficient and applicability in exact ISO 5167-2 calculation.”

Features:

- live sliders
- reset to demo case
- export chart data as CSV
- save scenario

---

## 12. Uncertainty Lab

Create:

```text
src/pages/UncertaintyLabPage.tsx
src/lib/formulas/uncertainty.ts
```

Inputs:

- δC/C
- δε/ε
- δd/d
- δD/D
- δΔp/Δp
- δρ/ρ
- additional installation uncertainty
- DP transmitter uncertainty as %span
- density source uncertainty

Output:

- combined relative uncertainty
- absolute uncertainty in qm
- absolute uncertainty in qv
- contribution chart
- final result:
  - q = value ± uncertainty
  - q within X %

Implement practical RSS-style uncertainty with special handling:

- installation additional uncertainty added arithmetically if selected
- show formula breakdown in UI
- allow user to toggle “educational simplified mode” vs “ISO-style component mode”

---

## 13. Fluid Properties Module

Create simple built-in presets:

```ts
export const fluidPresets = [
  {
    name: "Water at 20°C",
    fluidType: "liquid",
    densityKgM3: 998.2,
    dynamicViscosityPaS: 0.001002
  },
  {
    name: "Water at 25°C",
    fluidType: "liquid",
    densityKgM3: 997.0,
    dynamicViscosityPaS: 0.00089
  },
  {
    name: "Air, approximate",
    fluidType: "gas",
    densityKgM3: 1.204,
    dynamicViscosityPaS: 0.0000181,
    isentropicExponent: 1.4
  },
  {
    name: "Custom",
    fluidType: "custom"
  }
];
```

Add warning:

- fluid presets are approximate
- for final design, use verified properties at actual pressure and temperature

---

## 14. Unit Conversion

Create unit conversion for:

Flow:

- kg/s
- kg/h
- m³/s
- m³/h
- L/min
- GPM

Pressure:

- Pa
- kPa
- bar
- psi
- mmH2O
- inH2O

Diameter:

- m
- mm
- inch

Density:

- kg/m³
- specific gravity

Viscosity:

- Pa·s
- cP

Temperature:

- °C
- K
- °F

Make all calculations internally SI.

---

## 15. Report and Export

Create:

```text
src/pages/ReportPage.tsx
src/lib/report/buildReport.ts
```

Report should include:

- project title
- date/time
- calculation mode
- fluid properties
- pipe geometry
- orifice geometry
- tapping type
- input values
- calculated values
- assumptions
- warnings
- compliance status
- uncertainty summary
- iteration history, if applicable
- chart data summary
- disclaimer

Export options:

- JSON
- CSV
- printable HTML
- browser print to PDF

Do not require server-side PDF generation.

---

## 16. Local Presets

Use localStorage or IndexedDB.

Features:

- save scenario
- duplicate scenario
- delete scenario
- load scenario
- export/import JSON scenario file

Create:

```ts
export interface SavedScenario {
  id: string;
  name: string;
  createdAt: string;
  updatedAt: string;
  input: OrificeCalculationInput;
  result?: OrificeCalculationResult;
}
```

---

## 17. UI/UX Style

Design language:

- modern industrial engineering dashboard
- clean, serious, technical
- light/dark mode
- status badges
- equation panels
- compact input cards
- live charts
- animated SVG diagrams

Visual identity:

- App name: Rekalibrasi ISO 5167 Lab
- Accent: blue/cyan/amber engineering theme
- Avoid childish or game-like visuals
- Use clear units everywhere

Suggested layout:

```text
┌─────────────────────────────────────────────────────────────┐
│ Rekalibrasi ISO 5167 Lab      [Scenario] [Export] [Theme]   │
├───────────────┬─────────────────────────────┬───────────────┤
│ Navigation    │ Main Calculator / Simulator │ Notes/Warnings│
│               │                             │               │
│ Dashboard     │ Input cards                 │ Formula       │
│ Orifice Calc  │ Result cards                │ Assumptions   │
│ Tapping       │ SVG/Charts                  │ Compliance    │
│ Installation  │ Iteration table             │ References    │
│ Simulation    │                             │               │
│ Uncertainty   │                             │               │
└───────────────┴─────────────────────────────┴───────────────┘
```

---

## 18. Engineering Warning Engine

Create automatic warnings:

```ts
export function validateOrificeInput(input: OrificeCalculationInput): ValidationResult;
```

Warnings:

- beta <= 0 or beta >= 1
- d >= D
- density <= 0
- viscosity <= 0
- DP <= 0
- pipe not running full
- fluid not single phase
- gas pressure ratio too low if compressible mode is used
- pulsation flag enabled
- missing ISO 5167-2 exact coefficient model
- tapping position conceptual only
- installation table requires ISO 5167-2 verification
- temperature correction not applied
- primary device condition unknown
- edge condition worn/unknown
- pipe roughness unknown
- flow conditioner selected without compliance verification

Each warning should have:

- severity: info/warning/error
- explanation
- recommended action

---

## 19. Testing Requirements

Create unit tests for formula engine using Vitest.

Test:

- beta calculation
- flow calculation from known manual C and epsilon
- qv = qm / rho
- Reynolds calculation
- DP reverse calculation
- bore reverse calculation
- invalid beta handling
- uncertainty calculation
- unit conversion

Add sample test case:

```ts
D = 0.1 m
 d = 0.06 m
beta = 0.6
deltaP = 25000 Pa
rho = 997 kg/m3
mu = 0.00089 Pa.s
C = 0.606
epsilon = 1
```

Check:

- result is finite
- q > 0
- ReD > 0
- warnings do not include fatal error

---

## 20. GitHub Pages Deployment

Configure Vite for GitHub Pages.

Use HashRouter.

In `vite.config.ts`, set:

```ts
export default defineConfig({
  plugins: [react()],
  base: "/iso-5167-lab/"
});
```

Create GitHub Actions workflow:

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - name: Install
        run: npm ci

      - name: Test
        run: npm run test -- --run

      - name: Build
        run: npm run build

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./dist

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy
        id: deployment
        uses: actions/deploy-pages@v4
```

Package scripts:

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "test": "vitest"
  }
}
```

---

## 21. Required Folder Structure

Generate the project with this structure:

```text
iso-5167-lab/
  index.html
  package.json
  vite.config.ts
  tsconfig.json
  tailwind.config.ts
  postcss.config.js
  .github/
    workflows/
      deploy.yml
  src/
    main.tsx
    App.tsx
    routes.tsx
    styles/
      globals.css
    components/
      layout/
        AppShell.tsx
        Sidebar.tsx
        TopBar.tsx
      common/
        NumberInputWithUnit.tsx
        ResultCard.tsx
        StatusBadge.tsx
        WarningList.tsx
        FormulaPanel.tsx
        ExportButton.tsx
      flow/
        TappingLayoutSvg.tsx
        OrificePlateSvg.tsx
        FlowConditionerSvg.tsx
        IterationTable.tsx
        SensitivityChart.tsx
    pages/
      DashboardPage.tsx
      OrificeCalculatorPage.tsx
      TappingSimulatorPage.tsx
      InstallationCheckerPage.tsx
      FlowSimulationPage.tsx
      UncertaintyLabPage.tsx
      FlowConditionerAdvisorPage.tsx
      FluidPropertiesPage.tsx
      ReportPage.tsx
      ReferenceNotesPage.tsx
      SettingsPage.tsx
    lib/
      formulas/
        orificeIso5167.ts
        reynolds.ts
        uncertainty.ts
        unitConversion.ts
        iterativeSolver.ts
      validation/
        validateOrificeInput.ts
      installation/
        installationRules.ts
      report/
        buildReport.ts
      storage/
        scenarios.ts
    data/
      fluidPresets.ts
      roughnessPresets.ts
      conditionerPresets.ts
    store/
      useScenarioStore.ts
    tests/
      orificeIso5167.test.ts
      reynolds.test.ts
      uncertainty.test.ts
```

---

## 22. Acceptance Criteria

The project is complete when:

1. It runs with:

   ```bash
   npm install
   npm run dev
   ```

2. It builds with:

   ```bash
   npm run build
   ```

3. It deploys to GitHub Pages using GitHub Actions.

4. User can calculate orifice flow from ΔP with manual C and epsilon.

5. User can solve ΔP from target flow.

6. User can solve orifice bore/beta from target flow using iterative solver.

7. User can visualize corner, flange, and D-D/2 tapping conceptually.

8. User can run sensitivity simulations and see live charts.

9. User can estimate uncertainty.

10. User can save/load local scenarios.

11. User can export JSON/CSV/printable report.

12. UI clearly labels educational/default values and non-full-compliance states.

13. The app does not falsely claim full ISO 5167-2 compliance unless exact ISO 5167-2 equations, limits, and tables are implemented and tested.

14. The app includes clear engineering disclaimers.

---

## 23. Implementation Priority

Build in this order:

### Phase 1 — Static App Foundation

- Vite + React + TypeScript
- Tailwind
- routing
- layout
- theme
- dashboard

### Phase 2 — Formula Engine

- unit conversion
- beta
- flow equation
- Reynolds
- qv/qm
- validation

### Phase 3 — Orifice Calculator

- input forms
- result cards
- warnings
- formula panel

### Phase 4 — Iterative Solver

- solve DP
- solve d
- solve beta
- iteration history

### Phase 5 — Tapping Simulator

- SVG pipe
- taps
- flow arrows
- tapping type switcher
- compliance notes

### Phase 6 — Simulation

- ΔP vs Q
- β sensitivity
- density sensitivity
- Re trend

### Phase 7 — Installation & Conditioner

- installation checklist
- conditioner advisor
- pressure loss across conditioner

### Phase 8 — Uncertainty & Report

- uncertainty calculator
- chart contribution
- export
- scenario storage

### Phase 9 — Tests & GitHub Pages

- unit tests
- build fix
- deploy workflow

---

## 24. Final Instruction

Generate clean, production-ready code. Prioritize correctness, clarity, and maintainability. Keep all engineering formulas isolated in pure TypeScript functions so they can be tested independently from UI components. Add comments only where they explain engineering assumptions or compliance limitations.

Do not create a fake backend. Do not require environment variables. Do not use external paid APIs. Do not embed copyrighted ISO tables or long standard text. Use original explanations and clear references to the need for the official standard for final design.

The final result must be a static, interactive, educational engineering application deployable on GitHub Pages.
