
const fetch = require('node-fetch');
const url = `https://c1-aicoe-nodered-lb.prod.c1conversations.io/FabricWorkflow/api//exposed/chord/getApptSlots`;

try {
    const response = await fetch(url, {
        method: 'GET',
        headers: {
            'Content-Type': 'application/json'
        }
        })
    
    
    const result = await response.json();

    if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }
    return JSON.stringify(result);
} catch (error) {
    console.error('getDynamicMenu error:', error);
    return JSON.stringify({
        "error": "getApptSlots Error: " + error,
        "message": `Could not fetch data for ${$requestType}`
    });
}