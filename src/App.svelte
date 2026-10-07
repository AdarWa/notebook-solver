<script lang="ts">
  import {
    FileUploader,
    TextArea,
    Button,
    CodeSnippet,
    Grid,
    Row,
    Column
  } from "carbon-components-svelte";
  import "carbon-components-svelte/css/white.css";

  let files: ReadonlyArray<File> = [];
  let notebook: any = null;
  let taskDescription: string = "Fix the missing code in the following cells. Return the modified code wrapped in the exact same BEGIN and END markers. Do not change, add or remove any existing comments. Do not modify basic structure of the code.";
  let llmResponse: string = "";
  let fileName: string = "solved_notebook.ipynb";
  let copyButtonText: string = "Copy Task & All Cells";

  interface CodeCell {
    id: number;
    source: string;
    wrappedSource: string;
  }

  let codeCells: CodeCell[] = [];

  $: if (files.length > 0) {
    processFile(files[0]);
  }

  async function processFile(file: File) {
    fileName = file.name.replace(".ipynb", "_solved.ipynb");
    const text = await file.text();
    notebook = JSON.parse(text);

    codeCells = [];
    
    notebook.cells.forEach((cell: any, index: number) => {
      if (cell.cell_type === "code") {
        const sourceCode = Array.isArray(cell.source) ? cell.source.join("") : cell.source;
        const wrappedSource = `# --- BEGIN CELL ${index} ---\n${sourceCode}\n# --- END CELL ${index} ---`;
        codeCells = [...codeCells, { id: index, source: sourceCode, wrappedSource }];
      }
    });
  }

  // Concatenate the task definition and all extracted code blocks for easy pasting into an LLM
  async function copyAllToClipboard() {
    if (codeCells.length === 0) return;

    const allCellsText = codeCells.map(cell => cell.wrappedSource).join("\n\n");
    const fullPrompt = `${taskDescription}\n\n${allCellsText}`;

    try {
      await navigator.clipboard.writeText(fullPrompt);
      copyButtonText = "Copied!";
      setTimeout(() => {
        copyButtonText = "Copy Task & All Cells";
      }, 2000);
    } catch (err) {
      console.error("Failed to copy text: ", err);
      copyButtonText = "Failed to copy";
    }
  }

  function stitchNotebook() {
    if (!notebook || !llmResponse) return;

    const newNotebook = JSON.parse(JSON.stringify(notebook));
    
    const regex = /# --- BEGIN CELL (\d+) ---\n([\s\S]*?)\n?# --- END CELL \1 ---/g;
    let match;

    while ((match = regex.exec(llmResponse)) !== null) {
      const cellId = parseInt(match[1], 10);
      const newCode = match[2];

      if (newNotebook.cells[cellId] && newNotebook.cells[cellId].cell_type === "code") {
        const lines = newCode.split("\n").map(line => line + "\n");
        if (lines.length > 0) {
          lines[lines.length - 1] = lines[lines.length - 1].replace(/\n$/, "");
        }
        newNotebook.cells[cellId].source = lines;
      }
    }

    const blob = new Blob([JSON.stringify(newNotebook, null, 2)], { type: "application/json" });
    const url = URL.createObjectURL(blob);
    const anchor = document.createElement("a");
    anchor.href = url;
    anchor.download = fileName;
    document.body.appendChild(anchor);
    anchor.click();
    document.body.removeChild(anchor);
    URL.revokeObjectURL(url);
  }
</script>

<Grid>
  <Row>
    <Column>
      <br />
      <h2>Notebook Solver</h2>
      <br />
      <FileUploader
        labelTitle="Upload Jupyter Notebook"
        buttonLabel="Select .ipynb"
        accept={[".ipynb"]}
        bind:files
        status="complete"
      />
    </Column>
  </Row>

  {#if notebook}
    <Row>
      <Column>
        <br />
        <TextArea
          labelText="Task Description"
          bind:value={taskDescription}
          placeholder="Define the task"
          rows={3}
        />
      </Column>
    </Row>

    <Row>
      <Column>
        <br />
        <h3>Extracted Code Cells</h3>
        <br />
        <Button kind="secondary" on:click={copyAllToClipboard}>
          {copyButtonText}
        </Button>
        <br />
      </Column>
    </Row>

    {#each codeCells as cell}
      <Row>
        <Column>
          <br />
          <CodeSnippet type="multi" code={cell.wrappedSource} />
        </Column>
      </Row>
    {/each}

    <Row>
      <Column>
        <br />
        <h3>Stitch Results</h3>
        <br />
        <TextArea
          labelText="LLM Output"
          bind:value={llmResponse}
          placeholder="Paste the raw LLM response here"
          rows={10}
        />
        <br />
        <Button on:click={stitchNotebook}>Stitch & Download Notebook</Button>
        <br /><br />
      </Column>
    </Row>
  {/if}
</Grid>