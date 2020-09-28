<script>
	import { Jumbotron, Label, Input, FormGroup, Button, Col, Container, Row } from 'sveltestrap';
	import parser from 'fast-xml-parser';

	let mavenText = `<dependency>
    <groupId>org.jsoup</groupId>
    <artifactId>jsoup</artifactId>
    <version>1.13.1</version>
    <classifier>apis</classifier>
</dependency>`.trim();
	let gradleText = '';

	function gradleUpdate() {
		console.log('gradle update!');
	}

	function mavenUpdate() {
		let xml = parser.parse(mavenText);
		let deps = xml.dependency instanceof Array ? xml.dependency : [xml.dependency];

		gradleText = deps.map((dep) => {
			return `implementation("${dep.groupId}:${dep.artifactId}:${dep.version}")`
		}).join("\n");
	}

	mavenUpdate();

	//export let name;
</script>

<main>
	<Jumbotron class="text-center">
		<h1 class="display-3">Dependencies Converter</h1>
		<p class="lead">For Gradle & Maven (two-way)</p>
	</Jumbotron>
	<Container>
		<Row>
			<Col>
				<FormGroup>
					<Label for="mavenText"><strong>Maven</strong></Label>
					<p>&lt;dependencies&gt;<p>
					<Input type="textarea" name="text" id="mavenText" class="codeText" bind:value={mavenText} on:keyup={mavenUpdate}/>
					<p>&lt;/dependencies&gt;</p>
				</FormGroup>
			</Col>
			<Col>
				<FormGroup>
					<Label for="gradleText"><strong>Gradle</strong></Label>
					<p>dependencies &lbrace;<p>
					<Input type="textarea" name="text" id="gradleText" class="codeText" bind:value={gradleText} on:keyup={gradleUpdate}/>
					<p>&rbrace;</p>
				</FormGroup>
			</Col>
		</Row>
	</Container>
</main>