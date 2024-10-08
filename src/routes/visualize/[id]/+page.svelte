<script>
	import { page } from '$app/stores'; // Import the page store
	import { onMount } from 'svelte';
	import { sparql_url } from '../../../store';
	import { basicAuth } from '../../../store';
	import { subgraphTemplate } from '$lib/queries/subgraph.sparql.njk';

	import nunjucks from 'nunjucks';
	nunjucks.configure({ autoescape: true });
	import { Parser } from 'n3';

	import cytoscape from 'cytoscape';
	import cytoscapePopper from 'cytoscape-popper';
	import { computePosition, flip, shift, limitShift } from '@floating-ui/dom';
	import tippy from 'tippy.js';
	import 'tippy.js/themes/light-border.css';

	//let cy;
	let cyElements = []; // Will hold the nodes and edges for Cytoscape
	let serverResponse = ''; // Variable to hold the server's response
	let responseText = '';
	let hasError = false;

	const handleConnect = async (query) => {
		console.log('handleConnect with query:', query);

		try {
			// Reset error state and response text
			hasError = false;
			responseText = 'Catch data...';

			// Send the request using fetch
			const response = await fetch($sparql_url, {
				method: 'POST',
				headers: {
					'Content-Type': 'application/x-www-form-urlencoded',
					Authorization: `Basic ${$basicAuth}`,
					Accept: 'text/turtle'
				},
				body: new URLSearchParams({
					query: query
				})
			});

			if (response.ok) {
				// Parse the JSON response
				const data = await response.text();
				serverResponse = data;
				visualizeWorkflow(data);
			} else {
				console.error('SPARQL query failed:', response.statusText);
				throw new Error(`HTTP error! Status: ${response.status}`);
			}
		} catch (error) {
			console.error('Error:', error);
			hasError = true;
			serverResponse = `Fetching data failed.\nEndpoint URL: ${sparql_url}\nError message: ${error.message}\nPossible reasons: Wrong URL, Dataset, Endpoint, Username, or Password.`;
		}
	};

	// Automatically run the function when the component is mounted
	onMount(() => {
		const executionId = $page.params.id;
		const query = nunjucks.renderString(subgraphTemplate, { executionId });
		handleConnect(query);
	});

	// Parse the Turtle data and prepare Cytoscape elements
	async function visualizeWorkflow(data) {
		const parser = new Parser({ format: 'text/turtle' });

		const nodeMap = new Map(); // To store nodes and their properties

		/**
		 * @type {{ data: { source: any; target: any; label: any; }; }[]}
		 */
		const cyEdges = []; // To store edges (predicates as links between subject and object)

		// Some predicates simply define an edge between two nodes
		const edgeDefiningPreds = [
			'provone:wasPartOf',
			'prov:agent',
			'prov:hadPlan',
			'prov:qualifiedAssociation',
			'prov:used',
			'prov:wasGeneratedBy',
			'prov:wasAssociatedWith',
			'provone:controls'
		];

		const quads = parser.parse(data);

		quads.forEach((quad) => {
			const localName = getLocalName(quad.predicate.value);
			const subject = quad.subject.value;
			const object = quad.object.value;
			const predicate = quad.predicate.value;

			// Handle the predicate
			// Some predicates simply define an edge in the graph
			if (edgeDefiningPreds.includes(predicate)) {
				cyEdges.push({
					data: {
						source: subject,
						target: object,
						label: predicate
					}
				});
			} else {
				if (!nodeMap.has(quad.subject.value)) {
					nodeMap.set(quad.subject.value, {});
				}
				const nodeData = nodeMap.get(quad.subject.value);
				nodeData[localName] = quad.object.value;
			}
		});

		// Convert Map to Cytoscape node elements
		const cyNodes = Array.from(nodeMap.entries()).map(([id, data]) => ({
			data: {
				id,
				...data
			}
		}));

		// Add edges to the elements list
		cyElements = [...cyNodes, ...cyEdges];

		// Specify types
		//const agentTypes = {};

		// Parse the Turtle string and store each quad
		const cy = cytoscape({
			container: document.getElementById('cy'),
			elements: cyElements,
			style: [
				// Agent node style
				{
					selector: 'node[type="prov:Agent"], node[type="provone:User"]',
					style: {
						shape: 'polygon',
						'shape-polygon-points': '1, 1, -1 1, -1, 0, 0, -1, 1, 0',
						width: '40px',
						height: '40px',
						'background-color': '#e27602',
						'border-width': '1px',
						'border-color': 'black'
					}
				},
				{
					selector: 'node',
					style: {
						content: 'data(label)',
						label: 'data(label)'
					}
				},
				// Entity node Style
				{
					selector: 'node[type="prov:Entity"], node[type="provone:Controller"], node[type="provone:Data"], node[type="provone:Program"], node[type="provone:Workflow"]', 
					style: {
						shape: 'ellipse',
						width: '75px',
						height: '40px',
						'background-color': '#fffc87',
						'border-width': '1px',
						'border-color': '#353aff'
					}
				},
				// Activity node Style
				{
					selector: 'node[type="prov:Activity"], node[type="provone:Execution"]',
					style: {
						shape: 'rectangle',
						width: '40px',
						height: '40px',
						'background-color': '#9fb1fc',
						'border-width': '1px',
						'border-color': '#353aff'
					}
				},
				// Influence node Style
				{
					selector: 'node[type="prov:qualifiedAssociation"]',
					style: {
						shape: 'rectangle',
						width: '40px',
						height: '40px',
						'background-color': '#9fb1fc',
						'border-width': '1px',
						'border-color': '#353aff'
					}
				},
				{
					selector: 'edge',
					style: {
						width: 3,
						'line-color': '#ccc',
						'target-arrow-color': '#ccc',
						'target-arrow-shape': 'triangle',
						'curve-style': 'bezier',
						label: 'data(label)'
					}
				}
			],

			layout: {
				name: 'breadthfirst'
			}
		});

		cy.center();
		cy.fit();

		// Set tooltip text for each node
		cy.nodes().forEach(setTooltip);

		/**
		 * Mouseover
		 *
		 * Show node data information during mouseover
		 */

		// Mouseover
		cy.on('mouseover', 'node', function (evt) {
			var node = evt.target;

			console.log(node.data()['type']);
			node.scratch('_tooltip').show();
		});

		// Mouseout
		cy.on('mouseout', 'node', function (evt) {
			var node = evt.target;
			node.scratch('_tooltip').hide();
		});

		/**
		 * Mouse Click
		 * Make nodes and edges not connected to clicked node opaque.
		 */

		// Click
		cy.on('click', function (event) {
			// Remove highlights from previous clicks
			cy.nodes().removeStyle();
			cy.edges().removeStyle();

			var evtTarget = event.target;

			if (evtTarget == cy) {
				// This catches clicks somewhere in the graph
			} else if (evtTarget.isNode()) {
				var node = evtTarget;

				// Neighbors of clicked node
				var directlyConnected = node.neighborhood().merge(node);

				// Hide all nodes that are not in Neighborhood
				var absComplement = directlyConnected.absoluteComplement();

				absComplement.style('opacity', 0.2);
				directlyConnected.style('opacity', 1.0);

				// Highlight the clicked node
				node.style('border-width', '3px');
				node.style('border-color', 'red');
			} else if (evtTarget.isEdge) {
				// not yet implemented
			}
		});
	}

	function getLocalName(uri) {
		const parts = uri.split('#');
		if (parts.length > 1) {
			return parts[1];
		} else {
			return uri.split(':')[1];
		}
	}

	function popperFactory(ref, content, opts) {
		// see https://floating-ui.com/docs/computePosition#options
		const popperOptions = {
			// matching the default behaviour from Popper@2
			// https://floating-ui.com/docs/migration#configure-middleware
			middleware: [flip(), shift({ limiter: limitShift() })],
			...opts
		};

		function update() {
			computePosition(ref, content, popperOptions).then(({ x, y }) => {
				Object.assign(content.style, {
					left: `${x}px`,
					top: `${y}px`
				});
			});
		}
		update();
		return { update };
	}

	cytoscape.use(cytoscapePopper(popperFactory));

	/**
	 * Functions
	 */

	// Function that sets tipy tooltip for the element
	function setTooltip(ele) {
		var tooltipText = buildTooltipText(ele);

		// Add tippy tooltip to node data
		var tippyTooltip = makeTippy(ele, tooltipText);
		//tippyTooltip.hide();
		ele.scratch('_tooltip', tippyTooltip);
	}

	function makeTippy(ele, text) {
		var ref = ele.popperRef();

		// Since tippy constructor requires DOM element/elements, create a placeholder
		var dummyDomEle = document.createElement('div');

		var tip = tippy(dummyDomEle, {
			getReferenceClientRect: ref.getBoundingClientRect,
			trigger: 'manual', // mandatory
			// dom element inside the tippy:
			content: function () {
				// function can be better for performance
				var div = document.createElement('div');

				div.innerHTML = text;

				return div;
			},
			// your own preferences:
			theme: 'light-border',
			placement: 'bottom',
			hideOnClick: false,

			// if interactive:
			interactive: true,
			appendTo: document.body // or append dummyDomEle to document.body
		});

		return tip;
	}

	// Function returns tooltip text (HTML) for nodes and edges
	function buildTooltipText(ele) {
		var tooltipText = '';
		var data = ele.data();

		if (ele.isEdge()) {
			// empty
		}

		if (ele.isNode()) {
			var nodeType = data['type'];

			tooltipText = "";

			if (nodeType == 'provone:Controller') {
				tooltipText = 'Version: ' + data['softwareVersion'] + '</br>Build: ' + data['build'] + '</br>Timestamp: ' + data['timestamp'] + '</br>DSL: ' + data['dsl'];
			}

			if (nodeType == 'provone:User') {
				tooltipText =
					data['givenName'] +
					' ' +
					data['familyName'] +
					'</br>ORCID: ' +
					data['hasORCID'] +
					'</br>Mail: ' +
					data['mbox'];
			}

			if (nodeType == 'provone:Program') {
				tooltipText = "Script:</br>" + data['script'];
			}

			if (nodeType == 'provone:Workflow') {
				tooltipText = "Repository: " + data['repository'] + '</br>Revision: ' + data['revision'] + '</br>Location: ' + data['location'];
			}

			if (nodeType == 'activity') {
				tooltipText =
					'ID: ' +
					data['id'] +
					'</br>' +
					'Start: ' +
					data['startTime'] +
					'</br>' +
					'End: ' +
					data['endTime'];
			}

			if (nodeType == 'entity') {
				tooltipText =
					'ID: ' + data['id'] + '</br>SHA-256: ' + data['sha256'] + '</br>path: ' + data['path'];
			}

			if (nodeType == 'provone:Execution') {
				tooltipText =
					'Started: ' + data['startedAtTime'] + '</br>' + 'Ended:  ' + data['endedAtTime'];
			}
		}

		return tooltipText;
	}
</script>

<div>
	<h1>Execution Page</h1>
	<p>Execution ID: {$page.params.id}</p>
	<div id="cy"></div>
</div>

<style>
	#cy {
		position: absolute;
		left: 50px;
		width: calc(100% - 100px);
		height: calc(100% - 100px);
		border: 1px solid black;
		padding: 5px;
	}

	.tooltip {
		opacity: 0.5;
	}
</style>
