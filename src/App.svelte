<script>
	import { Alert, Jumbotron, Label, Input, FormGroup, Button, Col, Container, Row } from 'sveltestrap';
	import xmlParser from 'fast-xml-parser';
	const xmlComposer = new xmlParser.j2xParser({
		format: true,
		indentBy: "    ",
	});

	let alertText = '';

	let mavenInvalid = false;
	let mavenText = '';

	let gradleInvalid = false;
	let gradleText = `
implementation("org.jsoup:jsoup:$\{properties["jsoup.version"]}:apis")
testImplementation("org.junit.jupiter:junit-jupiter-api:5.4.2")
runtimeOnly("com.adobe.cq:core.wcm.components.content:2.11.1@zip")
	`.trim()

	function mavenParseXml(value) {
		if (!value) {
			throw new Error("Specify Maven XML!")
		}

		try {
			return xmlParser.parse(value);
		} catch (e) {
			throw new Error("Cannot parse Maven XML properly!")
		}
	}

	function mapMavenScope(value) {
		return {
			compile: 'implementation',
			runtime: 'runtimeOnly',
			provided: 'compileOnly',
			test: 'testImplementation',
		}[value] || 'implementation';
	}

	function mapMavenProperties(value) {
		const pattern = /\${([^}]+)}/g
		const matches = value.match(pattern)

		let result = value;
		if (matches) {
			matches.forEach(m => {
				const varName = m.substring(2, m.length - 1);
				result = result.replace(m, '${properties["'+ varName +'"]}');
			})
		}

		return result;
	}

	function mavenUpdate() {
		alertText = null;
		mavenInvalid = false

		try {
			let xml = mavenParseXml(mavenText);
			let deps = xml.dependency instanceof Array ? xml.dependency : [xml.dependency];

			gradleText = mapMavenProperties(deps.filter(dep => dep && dep.groupId && dep.artifactId && dep.version)
					.map(dep => {
						let scope = mapMavenScope(dep.scope);
						let result = `${dep.groupId}:${dep.artifactId}:${dep.version}`;
						if (dep.classifier) {
							result = `${result}:${dep.classifier}`
						}
						if (dep.type && dep.type !== 'jar') {
							result = `${result}@${dep.type}`;
						}
						return `${scope}("${result}")`
					}).join("\n"));
		} catch (e) {
			console.log(e)
			alertText = e.message;
			mavenInvalid = true
		}
	}

	function parseGradleLines(lines) {
		if (!lines) {
			throw new Error("Specify Gradle dependencies!")
		}

		return lines.split("\n")
				.map(l => l.trim())
				.filter(l => l && l.length)
				.map(l => parseGradleLine(l));
	}

	function parseGradleLine(line) {
		const patterns = [
			/"(\w+)"\("(.*)"\)/,
			/(\w+)\("(.*)"\)/,
		];

		let values = null;
		for (const pattern of patterns) {
			values = line.match(pattern);
			if (values != null) {
				return parseGradleDependency(values[1], values[2])
			}
		}
		throw new Error(`Cannot parse Gradle line '${line}'!`);
	}

	function parseGradleDependency(scope, value) {
		let type = null;
		if (value.includes("@")) {
			let parts = value.split("@");
			if (parts.length > 2) {
				throw new Error("Gradle dependency could not have more than one '@' character!")
			}

			value = parts[0];
			type = parts[1];
		}

		let parts = value.split(":")
		if (parts.length > 4) {
			throw new Error("Gradle dependency could not have more than four ':' characters!")
		}

		let groupId = parts[0];
		let artifactId = parts[1];
		let version = parts[2];
		let classifier = parts[3];
		scope = mapGradleScope(scope)

		return filterNulls({ groupId, artifactId, version, classifier, type, scope });
	}

	function mapGradleScope(value) {
		return {
			api: null,
			compile: null,
			compileOnly: 'provided',
			implementation: null,
			runtimeOnly: 'runtime',
			runtime: 'runtime',
			testApi: 'test',
			testImplementation: 'test'
		}[value];
	}

	function mapGradleProperties(value) {
		const pattern = /\${properties\["([^}]+)"\]}/g
		const matches = value.match(pattern)

		let result = value;
		if (matches) {
			matches.forEach(m => {
				const varName = m.substring(14, m.length - 3);
				result = result.replace(m, '${' + varName + '}');
			})
		}

		return result;
	}

	function filterNulls(obj) {
		return Object.entries(obj).reduce((a,[k,v]) => (v == null ? a : (a[k]=v, a)), {})
	}

	function gradleUpdate() {
		alertText = null;
		gradleInvalid = false;
		try {
			mavenText = mapGradleProperties(xmlComposer.parse({
				dependency: parseGradleLines(gradleText)
			}));
		} catch (e) {
			alertText = e.message;
			gradleInvalid = true;
		}
	}

	gradleUpdate();
</script>


<svelte:head>
	<title>Dependencies Converter for Gradle & Maven (two-way)</title>
</svelte:head>

<main>
	<Jumbotron class="text-center">
		<h1 class="display-4">Dependencies Converter</h1>
		<p class="lead">For Gradle & Maven (two-way)</p>
	</Jumbotron>
	<Container>
		<Row>
			<Col>
				<FormGroup>
					<Label for="mavenText"><strong>Maven</strong></Label>
					<p class="codeFont">&lt;dependencies&gt;<p>
					<Input readonly="false" type="textarea" wrap="off" name="text" id="mavenText" class="codeFont codeText" bind:invalid={mavenInvalid} bind:value={mavenText} on:keyup={mavenUpdate}/>
					<p class="codeFont">&lt;/dependencies&gt;</p>
				</FormGroup>
			</Col>
			<Col>
				<FormGroup>
					<Label for="gradleText"><strong>Gradle (Kotlin DSL)</strong></Label>
					<p class="codeFont">dependencies &lbrace;<p>
					<Input readonly="false" type="textarea" wrap="off" name="text" id="gradleText" class="codeFont codeText" bind:invalid={gradleInvalid} bind:value={gradleText} on:keyup={gradleUpdate}/>
					<p class="codeFont">&rbrace;</p>
				</FormGroup>
			</Col>
		</Row>
		{#if alertText}
			<Alert color="danger">
				{alertText}
			</Alert>
		{/if}
	</Container>
</main>