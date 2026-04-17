<script>
  import { onMount } from 'svelte';
  import { translations } from './lib/translations.js';
  // Source data (grams per 3000 g batch). We'll derive percents at runtime for accuracy.
  const SOURCE_GRAMS = {
    latte: {
      // atomized glucose base
      glucose: [
        { key: 'milk', name: 'Whole milk', grams3000: 1740 },
        { key: 'sucrose', name: 'Sugar (sucrose)', grams3000: 480 },
        { key: 'atomized_glucose', name: 'Atomized glucose', grams3000: 44 },
        { key: 'cream', name: 'Cream (35%)', grams3000: 600 },
        { key: 'skim_powder', name: 'Skim milk powder', grams3000: 120 },
        { key: 'lbg', name: 'Locust bean gum', grams3000: 16 },
      ],
      // glucose syrup base
      syrup: [
        { key: 'milk', name: 'Whole milk', grams3000: 1728 },
        { key: 'sucrose', name: 'Sugar (sucrose)', grams3000: 480 },
        { key: 'glucose_syrup', name: 'Glucose syrup', grams3000: 56 },
        { key: 'cream', name: 'Cream (35%)', grams3000: 600 },
        { key: 'skim_powder', name: 'Skim milk powder', grams3000: 120 },
        { key: 'lbg', name: 'Locust bean gum', grams3000: 16 },
      ],
    },
    uovo: {
      // atomized glucose base
      glucose: [
        { key: 'milk', name: 'Whole milk', grams3000: 1652 },
        { key: 'sucrose', name: 'Sugar (sucrose)', grams3000: 388 },
        { key: 'atomized_glucose', name: 'Atomized glucose', grams3000: 120 },
        { key: 'cream', name: 'Cream (35%)', grams3000: 540 },
        { key: 'skim_powder', name: 'Skim milk powder', grams3000: 132 },
        { key: 'yolk', name: 'Egg yolk', grams3000: 152 },
        { key: 'lbg', name: 'Locust bean gum', grams3000: 16 },
      ],
      // glucose syrup base
      syrup: [
        { key: 'milk', name: 'Whole milk', grams3000: 1636 },
        { key: 'sucrose', name: 'Sugar (sucrose)', grams3000: 448 },
        { key: 'glucose_syrup', name: 'Glucose syrup', grams3000: 76 },
        { key: 'cream', name: 'Cream (35%)', grams3000: 540 },
        { key: 'skim_powder', name: 'Skim milk powder', grams3000: 132 },
        { key: 'yolk', name: 'Egg yolk', grams3000: 152 },
        { key: 'lbg', name: 'Locust bean gum', grams3000: 16 },
      ],
    },
  };

  const INCREMENT = 750; // Standard gelato batch increment
  
  let savedState = {};
  if (typeof window !== 'undefined' && typeof localStorage !== 'undefined') {
    try {
      const saved = localStorage.getItem('gelacalc_state');
      if (saved) savedState = JSON.parse(saved);
    } catch(e) {}
  }

  let locale = savedState.locale || 'en';
  let style = (savedState.style && ['latte', 'uovo'].includes(savedState.style)) ? savedState.style : 'uovo';
  let base = (savedState.base && ['glucose', 'syrup'].includes(savedState.base)) ? savedState.base : 'syrup';
  let totalGrams = savedState.totalGrams || INCREMENT;
  let rows = [];
  let showRecipe = false;

  $: t = translations[locale] || translations['en'];

  $: {
    if (typeof window !== 'undefined' && typeof localStorage !== 'undefined') {
      localStorage.setItem('gelacalc_state', JSON.stringify({ locale, style, base, totalGrams }));
    }
  }

  const round1 = (n) => Math.round(n * 10) / 10;
  const round2 = (n) => Math.round(n * 100) / 100;

  function loadRecipe() {
    const baseRows = getBasePercents();
    rows = baseRows.map((r) => ({ ...r, grams: round1((r.percent / 100) * totalGrams) }));
  }

  function getBasePercents() {
    const source = SOURCE_GRAMS[style][base];
    const totalSource = source.reduce((a, r) => a + r.grams3000, 0);
    return source.map((r) => ({
      key: r.key,
      name: r.name,
      percent: totalSource > 0 ? (r.grams3000 / totalSource) * 100 : 0,
    }));
  }

  function recomputeFromPercents() {
    rows = rows.map((r) => ({ ...r, grams: round1((r.percent / 100) * totalGrams) }));
  }

  function recomputePercentsFromGrams() {
    const total = rows.reduce((a, r) => a + r.grams, 0);
    totalGrams = round1(total);
    rows = rows.map((r) => ({ ...r, percent: total > 0 ? (r.grams / total) * 100 : 0 }));
  }

  function proportionalAdjust(editedKey, newGrams) {
    // Always recalc from base proportions using the edited row as the anchor
    const clamped = Math.max(0, newGrams);
    const baseRows = getBasePercents();
    const baseEdited = baseRows.find((r) => r.key === editedKey);
    if (!baseEdited || baseEdited.percent <= 0) return;

    const newTotal = (clamped / baseEdited.percent) * 100;
    totalGrams = round1(newTotal);
    rows = baseRows.map((r) => ({
      ...r,
      grams: round1((r.percent / 100) * totalGrams),
    }));
  }

  function setBatchSize(newTotal) {
    totalGrams = newTotal;
    const baseRows = getBasePercents();
    rows = baseRows.map((r) => ({ ...r, grams: round1((r.percent / 100) * totalGrams) }));
    // Keep percents aligned with base recipe
    rows = rows.map((r, i) => ({ ...r, percent: baseRows[i].percent }));
  }

  function snap(n) {
    const v = Number(n);
    if (!isFinite(v)) return INCREMENT;
    return Math.max(INCREMENT, Math.round(v / INCREMENT) * INCREMENT);
  }

  function next(n) {
    const v = Number(n);
    if (!isFinite(v)) return INCREMENT;
    if (v % INCREMENT === 0) return v + INCREMENT;
    return Math.ceil(v / INCREMENT) * INCREMENT;
  }

  function prev(n) {
    const v = Number(n);
    if (!isFinite(v)) return INCREMENT;
    if (v % INCREMENT === 0) return Math.max(INCREMENT, v - INCREMENT);
    return Math.max(INCREMENT, Math.floor(v / INCREMENT) * INCREMENT);
  }

  onMount(loadRecipe);
</script>

<main class="container">
  <header style="display: flex; justify-content: space-between; align-items: flex-start;">
    <div>
      <h1>{t.title}</h1>
      <p class="subtitle">{t.subtitle}</p>
    </div>
    <select bind:value={locale} style="width: auto; margin-top: 16px;">
      <option value="en">English</option>
      <option value="fr">Français</option>
    </select>
  </header>

  <section class="controls">
    <div class="control">
      <label for="style">{t.style}</label>
      <select id="style" bind:value={style} on:change={loadRecipe}>
        <option value="latte">{t.styleLatte}</option>
        <option value="uovo">{t.styleUovo}</option>
      </select>
    </div>

    <div class="control">
      <label for="base">{t.sweetener}</label>
      <select id="base" bind:value={base} on:change={loadRecipe}>
        <option value="glucose">{t.atomizedGlucose}</option>
        <option value="syrup">{t.sweetSyrup}</option>
      </select>
    </div>

    <div class="control">
      <label for="batch">{t.batchSize}</label>
      <div class="servings">
        <button on:click={() => setBatchSize(prev(totalGrams))} aria-label="decrease">−{INCREMENT}g</button>
        <input id="batch" type="number" min="0" step="1" bind:value={totalGrams}
          on:input={(e) => setBatchSize(Math.max(0, Number(e.currentTarget.value) || 0))}
          on:change={(e) => setBatchSize(Math.max(0, Number(e.currentTarget.value) || 0))}>
        <button on:click={() => setBatchSize(next(totalGrams))} aria-label="increase">+{INCREMENT}g</button>
      </div>
    </div>
  </section>

  <section>
    <div class="summary">{t.totalBatch}: <strong>{Math.round(totalGrams)}</strong> g · {t.tubs}: <strong>{round1(totalGrams / INCREMENT)}</strong></div>
    <div class="table-wrap">
      <table>
        <thead>
          <tr>
            <th>{t.ingredient}</th>
            <th>{t.percent}</th>
            <th>{t.grams}</th>
          </tr>
        </thead>
        <tbody>
          {#each rows as row (row.key)}
            <tr>
              <td>{t[row.key] || row.name}</td>
              <td><span class="percent">{round1(row.percent)}%</span></td>
              <td>
                <input class="gram-input" type="number" min="0" step="1" value={row.grams}
                  on:input={(e) => proportionalAdjust(row.key, Number(e.currentTarget.value))}
                  on:change={(e) => proportionalAdjust(row.key, Number(e.currentTarget.value))}
                />
              </td>
            </tr>
          {/each}
        </tbody>
        <tfoot>
          <tr>
            <td>{t.total}</td>
            <td>{round1(rows.reduce((a, r) => a + r.percent, 0))}%</td>
            <td>{round1(rows.reduce((a, r) => a + r.grams, 0))}</td>
          </tr>
        </tfoot>
      </table>
    </div>
  </section>

  <div class="footnote">
    <small>{t.footnote}</small>
  </div>

  <section class="recipe-section">
    <button class="recipe-toggle" on:click={() => showRecipe = !showRecipe}>
      {showRecipe ? t.hideRecipe : t.showRecipe}
    </button>
    
    {#if showRecipe}
      <div class="recipe-content">
        <h3>{t.recipeTitle}</h3>
        
        <h4>{t.equipTitle}</h4>
        <ul>
          <li>{t.equip1}</li>
          <li>{t.equip2}</li>
          <li>{t.equip3}</li>
          <li>{t.equip4}</li>
          <li>{t.equip5}</li>
        </ul>
        
        <h4>{t.baseLatteTitle}</h4>
        <ol>
          <li>{@html t.baseLatte1}</li>
          <li>{@html t.baseLatte2}</li>
          <li>{@html t.baseLatte3}</li>
          <li>{@html t.baseLatte4}</li>
          <li>{@html t.baseLatte5}</li>
        </ol>
        
        <h4>{t.baseUovoTitle}</h4>
        <ol>
          <li>{@html t.baseUovo1}</li>
          <li>{@html t.baseUovo2}</li>
          <li>{@html t.baseUovo3}</li>
          <li>{@html t.baseUovo4}</li>
          <li>{@html t.baseUovo5}</li>
          <li>{@html t.baseUovo6}</li>
          <li>{@html t.baseUovo7}</li>
        </ol>
        
        <h4>{t.flavorStepsTitle}</h4>
        <ol>
          <li>{@html t.flavorSteps1}</li>
          <li>{@html t.flavorSteps2}</li>
          <li>{@html t.flavorSteps3}</li>
          <li>{@html t.flavorSteps4}</li>
          <li>{@html t.flavorSteps5}</li>
        </ol>
        
        <h4>{t.flavorVarTitle}</h4>
        <div class="flavor-grid">
          <div class="flavor-item">
            {@html t.flavorFiordilatte}
          </div>
          <div class="flavor-item">
            {@html t.flavorVanilla}
          </div>
          <div class="flavor-item">
            {@html t.flavorChocPre} {round1((37 * totalGrams) / 750)}{t.flavorChocPost} {totalGrams}{t.flavorChocEnd}
          </div>
          <div class="flavor-item">
            {@html t.flavorStracPre} {round1((60 * totalGrams) / 750)}{t.flavorStracPost}
          </div>
          <div class="flavor-item">
            {@html t.flavorHazelPre} {round1((37 * totalGrams) / 750)}{t.flavorHazelPost} {totalGrams}{t.flavorHazelEnd}
          </div>
          <div class="flavor-item">
            {@html t.flavorAlmondPre} {round1((60 * totalGrams) / 750)}{t.flavorAlmondPost} {totalGrams}{t.flavorAlmondEnd}
          </div>
          <div class="flavor-item">
            {@html t.flavorCoffeePre} {round1((15 * totalGrams) / 750)}{t.flavorCoffeePost} {totalGrams}{t.flavorCoffeeEnd}
          </div>
        </div>

        <h4>{t.tipsTitle}</h4>
        <ul>
          <li>{t.tip1}</li>
          <li>{t.tip2}</li>
          <li>{t.tip3}</li>
          <li>{t.tip4}</li>
          <li>{t.tip5}</li>
        </ul>
      </div>
    {/if}
  </section>
</main>

<footer class="site-footer">
  <a class="gh-link" href="https://github.com/HamletDuFromage/gelaratios" target="_blank" rel="noreferrer">
    <img class="gh-icon" src="/github-mark.svg" width="16" height="16" alt="GitHub" />
  </a>
</footer>
