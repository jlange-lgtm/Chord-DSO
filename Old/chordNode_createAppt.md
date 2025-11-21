
const fetch = require('node-fetch');
const url = `https://c1-aicoe-nodered-lb.prod.c1conversations.io/FabricWorkflow/api/exposed/chord/createAppt`;
const apptTime = $apptTime;
const operatoryId = $operatoryId;

if (!apptTime || apptTime.trim() === '') {
    return JSON.stringify({
        "error": {
            "message": "apptTime is required and cannot be empty",
            "type": "validation-error"
        }
    });
}

if (!operatoryId || operatoryId.trim() === '') {
    return JSON.stringify({
        "error": {
            "message": "operatoryId is required and cannot be empty. The operatoryId MUST be extracted from the 'operatory_id' field in the chordNode_getApptSlots tool response. You must call chordNode_getApptSlots first, extract the actual 'operatory_id' value from the selected appointment slot, and pass that exact value here.",
            "type": "validation-error"
        }
    });
}

if (operatoryId.startsWith('[') || operatoryId.includes('operatoryId') || operatoryId.includes('operatory_id') || operatoryId.toLowerCase().includes('extract') || operatoryId.toLowerCase().includes('from')) {
    return JSON.stringify({
        "error": {
            "message": "operatoryId must be the actual literal value from the 'operatory_id' field in chordNode_getApptSlots response, not a placeholder or description. Received invalid value: " + operatoryId + ". You MUST call chordNode_getApptSlots first, extract the exact 'operatory_id' value from the selected appointment slot, and pass that value here. NO EXCEPTIONS.",
            "type": "validation-error"
        }
    });
}

try {
    const response = await fetch(url, {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            apptTime: apptTime,
            operatoryId: operatoryId
        })
    });
    
    const contentType = response.headers.get('content-type');
    const responseText = await response.text();
    
    if (!response.ok) {
        let errorMessage = `HTTP ${response.status}: ${response.statusText}`;
        if (responseText) {
            try {
                const errorJson = JSON.parse(responseText);
                errorMessage = errorJson.message || errorJson.error || errorMessage;
            } catch (e) {
                errorMessage = `${errorMessage}. Response: ${responseText.substring(0, 200)}`;
            }
        }
        throw new Error(errorMessage);
    }
    
    if (!contentType || !contentType.includes('application/json')) {
        throw new Error(`Expected JSON response but received ${contentType || 'unknown content type'}. Response: ${responseText.substring(0, 200)}`);
    }
    
    const result = JSON.parse(responseText);
    return JSON.stringify(result);
} catch (error) {
    console.error('createAppointment error:', error);
    return JSON.stringify({
        "error": {
            "message": error.message || String(error),
            "type": error.name || "unknown-error"
        }
    });
}