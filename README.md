/**
 * Serverless Workflow Orchestrator (workflow_orchestrator.ts)
 *
 * Mini orchestrator that runs steps (functions) in order with retries and error handling.
 * Each step is a local async function simulating a serverless function (AWS Lambda).
 *
 * Usage:
 *  ts-node src/workflow_orchestrator.ts
 */
type Step = { name: string; run: () => Promise<any>; retries?: number };

class Orchestrator {
  steps: Step[];
  constructor(steps: Step[]) { this.steps = steps; }

  async runAll(context: Record<string, any> = {}) {
    for (const step of this.steps) {
      let attempts = 0;
      const max = step.retries ?? 2;
      while (attempts <= max) {
        try {
          console.log(`Running step ${step.name} (attempt ${attempts+1})`);
          const out = await step.run();
          context[step.name] = out;
          break;
        } catch (e) {
          console.error(`Step ${step.name} failed:`, e);
          attempts++;
          if (attempts > max) throw new Error(`Step ${step.name} failed after ${attempts} attempts`);
          await new Promise(r => setTimeout(r, 200 * attempts));
        }
      }
    }
    return context;
  }
}

// Demo functions
async function fetchUser() { return { id: 1, name: 'Alice' }; }
async function enrichUser() { return { enriched: true }; }
async function notify() { return 'notified'; }

(async function demo() {
  const steps: Step[] = [
    { name: 'fetchUser', run: fetchUser },
    { name: 'enrichUser', run: async () => { if (Math.random() < 0.2) throw new Error('transient'); return enrichUser(); }, retries: 3 },
    { name: 'notify', run: notify }
  ];
  const ork = new Orchestrator(steps);
  try {
    const res = await ork.runAll();
    console.log('Workflow completed:', res);
  } catch (e) {
    console.error('Workflow failed:', e);
  }
  process.exit(0);
})();
