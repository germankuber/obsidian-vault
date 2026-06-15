---
title: Reading Tracker
tags:
  - meta/reading-tracker
aliases:
  - Reading Tracker
  - Pendiente de leer
cssclasses:
  - reading-tracker
created: 2026-06-15
updated: 2026-06-15
---

# 📖 Reading Tracker

> [!info] Qué es esto
> Panel de **seguimiento de lectura** del vault. Cada nota guarda en su frontmatter un bloque `reading:` (`total_words`, `read_words`, `pct`, `last_read`). El pipeline `obsidian-knowledge-extractor` lo escribe al crear/actualizar notas; los skills `reading-status` / `reading-mark` lo consultan/actualizan. Este panel **lee ese mismo dato y lo muestra vivo** acá.
>
> - Una nota nueva nace en **0%** (todo sin leer). 🆕
> - Al agregarle contenido, `pct` baja y aparece como pendiente.
> - Al marcarla leída (`/reading-mark`), vuelve a **100%** y desaparece.

## ⏳ Pendiente de leer

```dataviewjs
// --- Pendiente de leer: agrupado por carpeta, barras █/░, botón "marcar leído" por nota ---
const BAR = 10;
const bar = (pct) => {
  const f = Math.round((pct ?? 0) / 100 * BAR);
  return "█".repeat(f) + "░".repeat(BAR - f);
};
const nf = (n) => n.toLocaleString("es-AR"); // separador de miles con punto
const today = () => window.moment().format("YYYY-MM-DD");

// Marca una nota como 100% leída escribiendo su frontmatter (API de Obsidian).
async function markRead(path, btn) {
  const file = app.vault.getAbstractFileByPath(path);
  if (!file) { new Notice("No encontré: " + path); return; }
  await app.fileManager.processFrontMatter(file, (fm) => {
    const r = fm.reading ?? {};
    const tot = r.total_words ?? 0;
    r.read_words = tot;
    r.pct = 100;
    r.last_read = today();
    fm.reading = r;
  });
  // feedback inmediato + refresco del panel
  if (btn) { btn.textContent = "✓ leído"; btn.disabled = true; }
  new Notice("✅ Marcado leído");
}

const pending = dv.pages()
  .where(p => p.reading && typeof p.reading.total_words === "number")
  .map(p => {
    const tot = p.reading.total_words ?? 0;
    const rd  = p.reading.read_words ?? 0;
    return { link: p.file.link, path: p.file.path, name: p.file.name,
             folder: p.file.folder || "(raíz)",
             pct: tot ? Math.round(rd / tot * 100) : 100,
             pend: tot - rd, isNew: rd === 0 };
  })
  .where(x => x.pend > 0);

if (pending.length === 0) {
  dv.paragraph("> [!success] ✅ Todo al día — no hay contenido sin leer.");
} else {
  const groups = {};
  for (const x of pending.array()) (groups[x.folder] ??= []).push(x);
  const totalNotes = pending.length;
  const totalWords = pending.array().reduce((s, x) => s + x.pend, 0);

  dv.paragraph(`> **${totalNotes} notas** · **${nf(totalWords)} palabras nuevas** sin consumir`);

  const folders = Object.entries(groups)
    .map(([f, arr]) => [f, arr, arr.reduce((s, x) => s + x.pend, 0)])
    .sort((a, b) => b[2] - a[2]);

  for (const [folder, arr, fw] of folders) {
    dv.header(3, `📁 ${folder} — ${arr.length} ${arr.length === 1 ? "nota" : "notas"} · ${nf(fw)} palabras`);
    arr.sort((a, b) => b.pend - a.pend);

    // tabla manual para poder meter un <button> real en cada fila
    const table = dv.container.createEl("table", { cls: "reading-pending" });
    const head = table.createEl("tr");
    ["Nota", "Progreso", "Pendiente", ""].forEach(h => head.createEl("th", { text: h }));

    for (const x of arr) {
      const tr = table.createEl("tr");
      // Nota (link clickeable — <a class="internal-link"> es lo que Obsidian resuelve)
      const tdNote = tr.createEl("td");
      const a = tdNote.createEl("a", { text: x.name.replace(/\.md$/, ""), cls: "internal-link" });
      a.dataset.href = x.path;
      a.setAttr("href", x.path);
      a.onclick = (e) => { e.preventDefault(); app.workspace.openLinkText(x.path, "", false); };
      if (x.isNew) tdNote.appendText(" 🆕");
      // Progreso
      tr.createEl("td").createEl("code", { text: `${bar(x.pct)} ${x.pct}%` });
      // Pendiente
      tr.createEl("td", { text: nf(x.pend) });
      // Botón
      const tdBtn = tr.createEl("td");
      const b = tdBtn.createEl("button", { text: "✅ Leído", cls: "mod-cta" });
      b.onclick = () => markRead(x.path, b);
    }
  }

  dv.paragraph(`---\n**Total: ${totalNotes} notas · ${nf(totalWords)} palabras nuevas**`);
  dv.paragraph(`_Tip: el botón ✅ marca esa nota al 100%. Para una categoría entera, desde el chat: \`/reading-mark <carpeta>\`._`);
}
```

## 📊 Tabla completa (todas las notas con tracking)

```dataview
TABLE WITHOUT ID
  file.link AS "Nota",
  reading.pct + "%" AS "Leído",
  (reading.total_words - reading.read_words) AS "Pendiente",
  reading.last_read AS "Última lectura"
WHERE reading.pct != null
SORT reading.pct ASC, (reading.total_words - reading.read_words) DESC
```

---

> [!tip] Cómo marcar algo como leído
> Desde el chat: `/reading-mark <nota o categoría>` (ej. `/reading-mark AI/Inference`, `/reading-mark "System Design"`). Eso pone esas notas en 100% y se actualizan acá automáticamente.
>
> Las notas que todavía **no** tienen bloque `reading:` no aparecen acá (están "sin tracking"). Para sentarlas en 100% como baseline, corré `/reading-mark` sobre la categoría una vez.
