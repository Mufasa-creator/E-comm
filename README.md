export default async function main(args) {
  // Destructuring to get orderId from inputVars
  const { VFAPIKey, userQuestion } = args.inputVars;
  
  // Check if orderId is provided
  if (!VFAPIKey) {
    return {
      next: { path: 'error' },
      trace: [{
        type: 'text',
        payload: { message: "Missing API Key" }
      }]
    };
  }

  const requestUrl = `https://general-runtime.voiceflow.com/knowledge-base/query`;
  const body = JSON.stringify({
        "chunkLimit": 3,
        "synthesis": false,
        "settings": {
          "model": "claude-instant-v1",
          "temperature": 0.1,
          "system": "You are a product recommendation assistant"
        },
        "tags": {
          "include": {
            "items": [
              "product-data"
            ]
          },
          "includeAllTagged": true
        },
        "question": userQuestion
          })

  try {
    // Making the GET request to the specified endpoint
    const response = await fetch(requestUrl, {
    method: 'POST',
    headers: {
      'Authorization': VFAPIKey,
      'accept': 'application/json',
      'content-type': 'application/json'
    },
    body: body
    });

    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }

    const data = response.json;
    let chunks = data.chunks;

    // Ensure the response contains order data
    if (!data) {
      throw new Error(`Missing data. ${data}`);
    }


  // Initialize the carousel object
  let carousel = {
      'layout': 'Carousel',
      'cards': []
  };
  
  // Assuming chunks is already an object array
  const kbChunks = chunks;
  chunks = JSON.stringify(chunks)
  
  // Select only the chunks with a score >= .75 and inventoryQuantity >= 1
  kbChunks.forEach((chunk) => {
      const { content, metadata } = chunk;
      const titleMatch = content.match(/title: (.*?);/);
      const bodyMatch = content.match(/body: (.*)/);
  
      if (titleMatch && bodyMatch) {  // Check if both title and body are found
          const title = metadata.title || "No title provided"; // Fallback title
          const bodyText = metadata.body.replace(/<[^>]*>/g, ''); // Removing HTML tags
          const trimmedBody = bodyText.length > 280 ? bodyText.substring(0, 277) + "..." : bodyText; // Limiting body to 280 characters
          const price = metadata.price;
          const stockStatus = metadata.inventoryQuantity > 0 ? `$${price}` : "(Out of Stock)";
  
          carousel.cards.push({
              imageUrl: metadata.mainImage || "", // Default image if not present
              title: `${title}: ${stockStatus}`, // Combining title with price or stock status
              description: {
                  text: trimmedBody
              },
              buttons: [
                  {
                      "name": "More info",
                      "request": {
                          "type": "click_URL",
                          "payload":{
                              "label":"Link",
                              "actions":[
                                  {
                                      "type":"open_url",
                                      "payload":{
                                          "url": metadata.productUrl
                                      }
                                  }
                              ]
                          }
                      }
                  }
              ]
          });
      }
  });
  
  let payload = JSON.stringify(carousel);

  // Return the carousel trace
  return {
    outputVars: { payload: payload },
    next: { path: 'success' },
    trace: [
      {
        type: "debug",
        payload: {message: payload}
      }
    ]
  };

  } catch (error) {
    return {
      next: { path: 'error' },
      trace: [{
        type: 'debug',
        payload: { message: `Failed to fetch order data: ${error.message}` }
      }]
    };
  }



}
