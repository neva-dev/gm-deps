<script>
	import { Jumbotron, Label, Input, FormGroup, Button, Col, Container, Row } from 'sveltestrap';
	import parser from 'fast-xml-parser';

	let mavenInvalid = false;
	let mavenText = `<dependency>
    <groupId>org.jsoup</groupId>
    <artifactId>jsoup</artifactId>
    <version>1.13.1</version>
    <classifier>apis</classifier>
</dependency>
<dependency>
	<groupId>org.junit.jupiter</groupId>
	<artifactId>junit-jupiter-api</artifactId>
	<version>5.4.2</version>
	<scope>test</scope>
</dependency>
 <dependency>
	<groupId>com.adobe.cq</groupId>
	<artifactId>core.wcm.components.content</artifactId>
	<version>2.11.1</version>
	<type>zip</type>
	<scope>runtime</scope>
</dependency>`.trim();

	let gradleInvalid = false;
	let gradleText = '';

	function mapMavenScope(value) {
		return {
			compile: 'implementation',
			runtime: 'runtimeOnly',
			provided: 'compileOnly',
			test: 'testImplementation',
		}[value] || 'implementation';
	}

	function mavenUpdate() {
		mavenInvalid = false
		try {
			let xml = parser.parse(mavenText);
			let deps = xml.dependency instanceof Array ? xml.dependency : [xml.dependency];

			gradleText = deps.filter(dep => dep.groupId && dep.artifactId && dep.version)
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
					}).join("\n");
		} catch (e) {
			mavenInvalid = true
		}
	}

	function parseGradleLines(lines) {
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
				return {
					scope: values[1],
					dependency: parseGradleDependency(values[2])
				}
			}
		}
		throw new Error(`Cannot parse Gradle line '${line}'!`);
	}

	function parseGradleDependency(value) {
		let extension = null;
		if (value.includes("@")) {
			let parts = value.split("@");
			value = parts[0];
			extension = parts[1];
		}

		let parts = value.split(":")
		let groupId = parts[0];
		let artifactId = parts[1];
		let version = parts[2];

		return { groupId, artifactId, version, extension };
	}

	function gradleUpdate() {
		gradleInvalid = false;
		try {
			console.log(parseGradleLines(gradleText));
		} catch (e) {
			gradleInvalid = true;
		}
	}

	mavenUpdate();
</script>

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
					<p>&lt;dependencies&gt;<p>
					<Input type="textarea" name="text" id="mavenText" class="codeText" bind:invalid={mavenInvalid} bind:value={mavenText} on:keyup={mavenUpdate}/>
					<p>&lt;/dependencies&gt;</p>
				</FormGroup>
			</Col>
			<Col>
				<FormGroup>
					<Label for="gradleText"><strong>Gradle</strong></Label>
					<p>dependencies &lbrace;<p>
					<Input type="textarea" name="text" id="gradleText" class="codeText" bind:invalid={gradleInvalid} bind:value={gradleText} on:keyup={gradleUpdate}/>
					<p>&rbrace;</p>
				</FormGroup>
			</Col>
		</Row>
	</Container>
</main>