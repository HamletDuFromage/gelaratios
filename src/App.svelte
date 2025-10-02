<script>
  import { onMount } from 'svelte';
  // Source data (grams per 3000 g batch). We'll derive percents at runtime for accuracy.
  const SOURCE_GRAMS = {
    latte: {
      // dextrose base
      glucose: [
        { key: 'milk', name: 'Whole milk', grams3000: 1740 },
        { key: 'sucrose', name: 'Sugar (sucrose)', grams3000: 480 },
        { key: 'dextrose', name: 'Dextrose', grams3000: 44 },
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
      // dextrose base
      glucose: [
        { key: 'milk', name: 'Whole milk', grams3000: 1652 },
        { key: 'sucrose', name: 'Sugar (sucrose)', grams3000: 448 },
        { key: 'dextrose', name: 'Dextrose', grams3000: 60 },
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
  
  let style = 'latte';
  let base = 'glucose';
  let totalGrams = INCREMENT;
  let rows = [];
  let showRecipe = false;

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
    // Preserve proportions: edited row set to new grams, others scaled proportionally.
    const clamped = Math.max(0, newGrams);
    const edited = rows.find((r) => r.key === editedKey);
    if (!edited) return;
    
    // If the edited ingredient was 0, restore from base recipe proportions
    if (edited.grams === 0 && clamped > 0) {
      const baseRows = getBasePercents();
      const baseEdited = baseRows.find((r) => r.key === editedKey);
      if (baseEdited) {
        // Calculate what the total should be based on the new value and base percentage
        const newTotal = (clamped / baseEdited.percent) * 100;
        rows = baseRows.map((r) => ({ ...r, grams: round1((r.percent / 100) * newTotal) }));
        totalGrams = newTotal;
        return;
      }
    }
    
    // Calculate the scale factor based on the ratio of new to old grams for the edited ingredient
    const scale = edited.grams > 0 ? clamped / edited.grams : 1;
    
    // Scale all other ingredients proportionally
    rows = rows.map((r) => 
      r.key === editedKey 
        ? { ...r, grams: round1(clamped) }
        : { ...r, grams: round1(r.grams * scale) }
    );
    
    // Update total and recompute percents from grams
    totalGrams = rows.reduce((a, r) => a + r.grams, 0);
    rows = rows.map((r) => ({ ...r, percent: totalGrams > 0 ? (r.grams / totalGrams) * 100 : 0 }));
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
  <header>
    <h1>Gelaratios</h1>
    <p class="subtitle">Perfect gelato ratios</p>
  </header>

  <section class="controls">
    <div class="control">
      <label for="style">Style</label>
      <select id="style" bind:value={style} on:change={loadRecipe}>
        <option value="latte">Gelato al latte</option>
        <option value="uovo">Gelato all'uovo</option>
      </select>
    </div>

    <div class="control">
      <label for="base">Sweetener</label>
      <select id="base" bind:value={base} on:change={loadRecipe}>
        <option value="glucose">Dextrose</option>
        <option value="syrup">Glucose syrup</option>
      </select>
    </div>

    <div class="control">
      <label for="batch">Batch size</label>
      <div class="servings">
        <button on:click={() => setBatchSize(prev(totalGrams))} aria-label="decrease">−{INCREMENT}g</button>
        <input id="batch" type="number" min="1" step="1" bind:value={totalGrams}
          on:input={(e) => setBatchSize(Math.max(1, Number(e.currentTarget.value) || 0))}
          on:change={(e) => setBatchSize(Math.max(1, Number(e.currentTarget.value) || 0))}>
        <button on:click={() => setBatchSize(next(totalGrams))} aria-label="increase">+{INCREMENT}g</button>
      </div>
    </div>
  </section>

  <section>
    <div class="summary">Total batch: <strong>{totalGrams}</strong> g · Tubs: <strong>{round2(totalGrams / INCREMENT)}</strong></div>
    <div class="table-wrap">
      <table>
        <thead>
          <tr>
            <th>Ingredient</th>
            <th>Percent</th>
            <th>Grams</th>
          </tr>
        </thead>
        <tbody>
          {#each rows as row (row.key)}
            <tr>
              <td>{row.name}</td>
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
            <td>Total</td>
            <td>{round1(rows.reduce((a, r) => a + r.percent, 0))}%</td>
            <td>{round1(rows.reduce((a, r) => a + r.grams, 0))}</td>
          </tr>
        </tfoot>
      </table>
    </div>
  </section>

  <div class="footnote">
    <small>Edit fields directly to fine-tune; others adjust proportionally.</small>
  </div>

  <section class="recipe-section">
    <button class="recipe-toggle" on:click={() => showRecipe = !showRecipe}>
      {showRecipe ? 'Hide' : 'Show'} Recipe Instructions
    </button>
    
    {#if showRecipe}
      <div class="recipe-content">
        <h3>Homemade Gelato Recipe</h3>
        
        <h4>Equipment Needed</h4>
        <ul>
          <li>Ice cream maker (compressor or freezer-bowl type)</li>
          <li>Digital thermometer</li>
          <li>Immersion blender</li>
          <li>Whisk and mixing bowls</li>
          <li>Pre-chilled storage container</li>
        </ul>
        
        <h4>Base Preparation - Milk-Based Gelato</h4>
        <ol>
          <li><strong>Mix dry ingredients:</strong> Combine sugar, milk powder, dextrose (or glucose syrup), and locust bean gum in a bowl</li>
          <li><strong>Heat liquids:</strong> Place milk and cream in a saucepan, heat to a light boil</li>
          <li><strong>Combine:</strong> Pour hot liquid over dry ingredients while whisking vigorously to prevent lumps</li>
          <li><strong>Cook:</strong> Return mixture to heat, bring to exactly 85°C (185°F) and maintain for 2 minutes</li>
          <li><strong>Cool:</strong> Remove from heat and let cool to room temperature</li>
        </ol>
        
        <h4>Base Preparation - Egg-Based Gelato</h4>
        <ol>
          <li><strong>Mix dry ingredients:</strong> Combine milk powder, dextrose (or glucose syrup), and locust bean gum</li>
          <li><strong>Prepare yolks:</strong> Beat sugar with egg yolks until pale and thick</li>
          <li><strong>Combine dry with yolks:</strong> Gently fold dry ingredients into yolk mixture (don't beat)</li>
          <li><strong>Heat liquids:</strong> Bring milk and cream to a boil</li>
          <li><strong>Temper eggs:</strong> Slowly pour hot liquid into yolk mixture while whisking constantly</li>
          <li><strong>Cook:</strong> Return to heat, bring to 85°C (185°F) and maintain for 2 minutes</li>
          <li><strong>Cool:</strong> Remove from heat and let cool to room temperature</li>
        </ol>
        
        <h4>Flavoring & Final Steps</h4>
        <ol>
          <li><strong>Add flavorings:</strong> Once base is cool, add your chosen flavoring ingredients</li>
          <li><strong>Blend:</strong> Use immersion blender for 2 minutes to emulsify and incorporate air</li>
          <li><strong>Mature:</strong> Refrigerate for 4-6 hours (overnight is best) to develop flavor and texture</li>
          <li><strong>Churn:</strong> Process in ice cream maker for 40-45 minutes until thick and creamy</li>
          <li><strong>Store:</strong> Transfer to pre-chilled container and freeze for 2-3 hours before serving</li>
        </ol>
        
        <h4>Popular Flavor Variations</h4>
        <div class="flavor-grid">
          <div class="flavor-item">
            <strong>Fiordilatte (Plain):</strong> Use the base without additions
          </div>
          <div class="flavor-item">
            <strong>Vanilla:</strong> Split 1 vanilla bean lengthwise, boil with milk, let steep 1 hour, strain before using
          </div>
          <div class="flavor-item">
            <strong>Chocolate:</strong> Add {round1((37 * totalGrams) / 750)}g unsweetened cocoa powder to {totalGrams}g base after cooking
          </div>
          <div class="flavor-item">
            <strong>Stracciatella:</strong> Add {round1((60 * totalGrams) / 750)}g melted dark chocolate while churning (last 2 minutes)
          </div>
          <div class="flavor-item">
            <strong>Hazelnut/Pistachio:</strong> Add {round1((37 * totalGrams) / 750)}g pure nut paste to {totalGrams}g base after cooking
          </div>
          <div class="flavor-item">
            <strong>Almond:</strong> Add {round1((60 * totalGrams) / 750)}g pure almond paste to {totalGrams}g base after cooking
          </div>
          <div class="flavor-item">
            <strong>Coffee:</strong> Add {round1((15 * totalGrams) / 750)}g instant coffee powder to {totalGrams}g base after cooking
          </div>
  </div>

        <h4>Tips for Success</h4>
        <ul>
          <li>Always use a thermometer - temperature is crucial for proper texture</li>
          <li>Pre-chill your storage container in the freezer for 30 minutes</li>
          <li>Don't skip the maturation step - it improves flavor and texture</li>
          <li>Churn until the mixture looks like soft-serve ice cream</li>
          <li>Store in an airtight container to prevent ice crystals</li>
        </ul>
      </div>
    {/if}
  </section>
</main>
