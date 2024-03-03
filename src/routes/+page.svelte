<script lang=ts>
    function hasScript(str: string) {
        return new DOMParser().parseFromString(str, 'text/html').querySelector('script') !== null;
    };

    function toTitleLine(line: string) {
        if (line.startsWith("# ")) {
            return line.slice(2).trimStart()
        }
    }

    function toCharacterLine(line: string) {
        if (line.startsWith("## ")) {
            return line.slice(3).trimStart()
        }
    }

    function toParentheticalLine(line: string) {
        if (line.startsWith("(")) {
            return line;
        }
    }

    var error: string | null = null;

    function markdownToScript(str: string) {
        if (hasScript(str)) {
            error = "Malicious";
            return;
        }
        error = null;
        var lines = str.split('\n').map(line => line.trim()).filter(line => line.length > 0).reverse();

        var title = lines.pop();
        if (title === undefined) {
            error = "NoContent";
            return;
        }
        title = toTitleLine(title);
        if (title === undefined) {
            error = "IncorrectTitle";
            return;
        }

        var finalScript = `<style>
html {
    font-size: 16px;
}
html, body {
    width: 210mm;
}
* {
    font-family: Courier, monospace;
    font-size: 1rem;
    margin: 0;
    padding: 0;
}
main {
    padding: 3rem;
    display: flex;
    flex-direction: column;
}
@media print {
    .pagebreak { page-break-before: always; }
    output { display: none; }
}
.character {
    align-self: center;
    text-align: center;
    margin-bottom: 0;
}
.paren {
    align-self: center;
    text-align: center;
    margin-bottom: 0;
}
.dialogue {
    align-self: center;
    max-width: 50%;
}
h3 {
    margin-top: 1rem;
    margin-bottom: 1rem;
    font-weight: bold;
    display: flex;
}
h1 {
    font-size: 2rem;
    align-self: center;
    text-align: center;
    margin-top: 13rem;
}
h2 {
    font-size: 1.5rem;
    font-weight: normal;
    align-self: center;
    text-align: center;
}
.preSceneNum {
    margin-right: 1.5rem;
}
.postSceneNum {
    text-align: right;
    padding-left: 1.5rem;
    flex-grow: 1;
}
p {
    margin-left: 2rem;
    margin-right: 2rem;
    margin-bottom: 1rem;
}
</style>
<main>
<output>
    For proper formatting, print this document.
</output>
    <h1>${title}</h1>`;
        const createScript = () => {
            var newTab = window.open();
            finalScript += "</main>";
            if (newTab) {
                newTab.document.body.innerHTML = finalScript;
                error = "Success";
            } else {
                error = "CannotOpenTab";
            }
        };

        const addPageBreak = () => {
            finalScript += "<div class=\"pagebreak\"></div>";;
        }

        var tmpLine = lines.pop();
        if (tmpLine === undefined) {
            createScript();
            return;
        }
        const subtitle = toTitleLine(tmpLine);
        if (subtitle) {
            finalScript += `<h2>${subtitle}</h2>`;
        }
        addPageBreak();

        var inCharacter = false;
        var sceneNum = 0;
        while (true) {
            tmpLine = lines.pop();
            if (tmpLine === undefined) {
                break;
            }
            const scene = toTitleLine(tmpLine);
            if (scene) {
                sceneNum += 1;
                finalScript += `<h3><span class="preSceneNum">${sceneNum}</span>${scene.toUpperCase()}<span class="postSceneNum">${sceneNum}</span></h3>`;
                inCharacter = false;
                continue;
            }
            const character = toCharacterLine(tmpLine);
            if (character) {
                finalScript += `<p class="character">${character.toUpperCase()}</p>`;
                inCharacter = true;
                continue;
            }
            if (inCharacter) {
                const paren = toParentheticalLine(tmpLine);
                if (paren) {
                    finalScript += `<p class="paren">${paren}</p>`;
                    continue;
                } else {
                    finalScript += `<p class="dialogue">${tmpLine}</p>`;
                }
            } else {
                finalScript += `<p>${tmpLine}</p>`;
            }
            inCharacter = false;
        }
        createScript();
    }

    var files: FileList;
</script>

<body>
    <header>
        <h1>MScript</h1>
        <p>A free to use, and simple, script writing software for films and shows.</p>
    </header>

    <main>
        <input type="file" bind:files={files}>
    <button on:click={() => {
        var reader = new FileReader();
        reader.readAsText(files[0], "UTF-8");
        reader.onload = function (evt) {
            const contents = evt.target?.result;
            if (typeof contents !== "string") {
                error = "NotText";
                return;
            }
            markdownToScript(contents);
        }
        reader.onerror = function () {
            error = "Unreadable";
        }
    }}>Submit</button>

    {#if error === null}
    <!-- Do nothing -->
    {:else if error === "Success"}
    <output id="success">The script was opened in a new tab!</output>
    {:else if error === "Unreadable"}
    <output>The file is unreadable. This could be due to browser restrictions.</output>
    {:else if error === "NotText"}
    <output>The uploaded file does not contain text.</output>
    {:else if error === "Malicious"}
    <output>The uploaded file may contain harmful code!</output>
    {:else}
    <output>Faced the following error without more details: {error}</output>
    {/if}
    </main>
    
</body>

<style>
    body, main {
        display: flex;
        flex-direction: column;
    }
    output {
        color: red;
    }

    #success {
        color: green;
    }

    * {
        font-family: Courier, monospace;
        font-size: 12px;
        margin:0px;
        padding:0px;
    }
</style>