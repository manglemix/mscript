<script lang="ts">
	function hasScript(str: string) {
		return new DOMParser().parseFromString(str, 'text/html').querySelector('script') !== null;
	}

	function toTitleLine(line: string) {
		if (line.startsWith('# ')) {
			return line.slice(2).trimStart();
		}
	}

	function toCharacterLine(line: string) {
		if (line.startsWith('## ')) {
			return line.slice(3).trimStart();
		}
	}

	function toParentheticalLine(line: string) {
		if (line.startsWith('(')) {
			return line;
		}
	}

	function toTransitionLine(line: string) {
		if (line.startsWith('### ')) {
			line = line.slice(4).trimStart();
			if (!line.endsWith(':')) {
				line += ':';
			}
			return line;
		}
	}

	var error: string | null = null;

	function markdownToScript(str: string) {
		if (hasScript(str)) {
			error = 'Malicious';
			return;
		}
		error = null;
		var lines = str
			.split('\n')
			.map((line) => line.trim())
			.filter((line) => line.length > 0)
			.reverse();

		var title = lines.pop();
		if (title === undefined) {
			error = 'NoContent';
			return;
		}
		title = toTitleLine(title);
		if (title === undefined) {
			error = 'IncorrectTitle';
			return;
		}

		var finalScript = `<style>
html {
    font-size: 16px;
    display: flex;
    flex-direction: column;
}
body {
    width: 210mm;
}
* {
    font-family: Courier, monospace;
    font-size: 1rem;
    margin: 0;
    padding: 0;
}
body {
    align-self: center;
}
main {
    padding: 3rem;
    display: flex;
    flex-direction: column;
}
.pagebreak {
    height: 5rem;
}
@media print {
    .pagebreak { page-break-before: always; height: 0; }
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
.transition {
    align-self: end;
    margin-right: 0.5rem;
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
<title>${title}</title>
<main>
<output>
    For proper formatting, print this document. Enable Headers and Footers to get page numbers and datetime.
</output>
    <h1>${title}</h1>`;
		const createScript = () => {
			var newTab = window.open();
			finalScript += '</main>';
			if (newTab) {
				newTab.document.body.innerHTML = finalScript;
				error = 'Success';
			} else {
				error = 'CannotOpenTab';
			}
		};

		const addPageBreak = () => {
			finalScript += '<div class="pagebreak"></div>';
		};

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
			const trans = toTransitionLine(tmpLine);
			if (trans) {
				finalScript += `<p class="transition">${trans.toUpperCase()}</p>`;
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

<svelte:head>
	<title>MScript</title>
</svelte:head>

<body>
	<header>
		<h1>MScript</h1>
		<p>A free to use, simple, script writing software for films and shows.</p>
	</header>

	<main>
		<input type="file" bind:files />
		<button
			on:click={() => {
				var reader = new FileReader();
				reader.readAsText(files[0], 'UTF-8');
				reader.onload = function (evt) {
					
					const contents = evt.target?.result;
					if (typeof contents !== 'string') {
						error = 'NotText';
						return;
					}
					markdownToScript(contents);
				};
				reader.onerror = function () {
					error = 'Unreadable';
				};
			}}>Submit</button
		>

		{#if error === null}
			<!-- Do nothing -->
		{:else if error === 'Success'}
			<output id="success">The script was opened in a new tab!</output>
		{:else if error === 'Unreadable'}
			<output>The file is unreadable. This could be due to browser restrictions.</output>
		{:else if error === 'IncorrectTitle'}
			<output>The uploaded file does not contain a valid title. Please review the example <a href="https://raw.githubusercontent.com/manglemix/mscript/main/static/example.md">source file</a>.</output>
		{:else if error === 'NoContent'}
			<output>The uploaded file is empty.</output>
		{:else if error === 'NotText'}
			<output>The uploaded file does not contain text.</output>
		{:else if error === 'Malicious'}
			<output>The uploaded file may contain harmful code!</output>
		{:else}
			<output>Faced the following error without more details: {error}</output>
		{/if}

		<h1>FAQ</h1>
		<ul>
			<li>
				<h2>How does this work?</h2>
				<p>
					Simply upload a text file that follows the markdown-script format, and this webapp will reformat it into industry-standard scripts.
					Here is an example <a href="https://raw.githubusercontent.com/manglemix/mscript/main/static/example.md">source file</a>. Empty lines are ignored, extra spaces are removed, capitalization is fixed, and missing
					punctuation will be added when possible. The only thing you need to pay attention to is the number of # you use.
				</p>
			</li>
			<li>
				<h2>Privacy?</h2>
				<p>
					This webapp runs completely on your web browser and no information is sent to anyone else. You can review the code <a href="https://github.com/manglemix/mscript">here</a>.
					Please open an issue if there is some functionality that is missing.
				</p>
			</li>
			<li>
				<h2>Why is this free with no ads?</h2>
				<p>
					Since this webapp does no server-side processing, it is completely free for me to deploy it. The repository is also public, which allows me to use Github Pages without paying.
				</p>
			</li>
			<li>
				<h2>Why does this exist?</h2>
				<p>
					A lot of the good script-writing software is paid, and most of the free ones are hard to find or have clunky UIs. A lot of these services are also bloated with other
					functionality more suited to full production. I just needed something for my classes that was easy to use and did not require me to research actual scripts to learn the format.
					Visit my <a href="https://www.manglemix.com/">website</a> to learn more about me.
				</p>
			</li>
		</ul>
	</main>
</body>

<style>
	body,
	main {
		display: flex;
		flex-direction: column;
	}
    main {
        margin-top: 2rem;
        gap: 0.5rem;
    }
	:global(html) {
		display: flex;
		flex-direction: column;
	}
    main button {
        width: min-content;
		padding: 0.35rem;
		font-weight: bold;
		border-radius: 0.5rem;
		font-size: 1.2rem;
    }
	main > h1 {
		margin-top: 3rem;
	}
	body {
		padding: 1rem;
    	width: 210mm;
		align-self: center;
	}
	output {
		color: red;
	}

	li {
		list-style-type: none;
		margin-bottom: 1rem;
	}

	#success {
		color: green;
	}

	* {
		font-family: Courier, monospace;
		margin: 0;
		padding: 0;
	}
</style>
