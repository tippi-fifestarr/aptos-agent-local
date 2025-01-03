# Setting Up Your First Aptos Agent - Part 2: Building Your AI Blockchain Assistant

Think of an AI blockchain agent like a new employee at a bank. They need two things to do their job: knowledge of banking procedures (that's our AI's instructions) and access to the actual banking systems (that's our blockchain integration). In Part 1, we set up our Python environment. Now, we'll create an AI assistant that can actually interact with the Aptos blockchain - giving it both the knowledge and the tools it needs to help users.

This tutorial continues from [Part 1](https://github.com/tippi-fifestarr/aptos-agent-local/tree/setup), where we set up our Python environment and virtual environment.

## Prerequisites
- Completed [Part 1 setup](https://github.com/tippi-fifestarr/aptos-agent-local/tree/setup)
- OpenAI API key (from platform.openai.com)
- In your activated virtual environment (`source venv/bin/activate`)
- Python 3.11 configured in venv local (check with `python --version`)

> [!NOTE]  
> Throughout this guide, we'll provide context for what each piece does and why we need it.

## 1. Environment Configuration

1. Ensure you're in your virtual environment:
```bash
source venv/bin/activate  # On Windows: .\venv\Scripts\activate
```

> [!NOTE]  
> You should see `(aptos-agent)` or similar at the start of your prompt

2. Create your `.env` file:
```bash
touch .env
```

3. Add your OpenAI API key:
```bash
echo "OPENAI_API_KEY=your-key-here" >> .env
```

> [!IMPORTANT]  
> Replace `your-key-here` with your actual OpenAI API key. Never commit this file to git!

## 2. Installing Dependencies

1. Update pip first:
```bash
python -m pip install --upgrade pip
```

2. Install Swarm and other dependencies:
```bash
pip install git+https://github.com/openai/swarm.git openai python-dotenv requests requests-oauthlib aptos-sdk
```

> [!NOTE]  
> Let's understand what each package does:
> - `swarm`: OpenAI's framework for creating AI agents that can use tools and make decisions
> - `openai`: Connects to OpenAI's API for the language model
> - `python-dotenv`: Loads environment variables (like your API keys)
> - `requests` & `requests-oauthlib`: Handles HTTP requests and OAuth authentication
> - `aptos-sdk`: Interfaces with the Aptos blockchain

3. Save the dependencies list:
```bash
pip freeze > requirements.txt
```

> [!NOTE]  
> Think of `requirements.txt` like `package-lock.json` in Node.js - it locks your dependency versions

## 3. Setting Up the Aptos SDK Wrapper

1. Create the SDK wrapper file:
```bash
touch aptos_sdk_wrapper.py
```

2. Add the Aptos integration code to `aptos_sdk_wrapper.py`:
```python
import os
from aptos_sdk.account import Account, AccountAddress
from aptos_sdk.async_client import FaucetClient, RestClient
from aptos_sdk.transactions import EntryFunction, TransactionArgument, TransactionPayload
from aptos_sdk.bcs import Serializer

# Initialize clients for testnet
rest_client = RestClient("https://api.testnet.aptoslabs.com/v1")
faucet_client = FaucetClient("https://faucet.testnet.aptoslabs.com",
                             rest_client)

async def fund_wallet(wallet_address, amount):
    """Funds a wallet with a specified amount of APT."""
    print(f"Funding wallet: {wallet_address} with {amount} APT")
    amount = int(amount)
    if amount > 1000:
        raise ValueError(
            "Amount too large. Please specify an amount less than 1000 APT")
    octas = amount * 10**8  # Convert APT to octas
    if isinstance(wallet_address, str):
        wallet_address = AccountAddress.from_str(wallet_address)
    txn_hash = await faucet_client.fund_account(wallet_address, octas, True)
    print(f"Transaction hash: {txn_hash}\nFunded wallet: {wallet_address}")
    return wallet_address

async def get_balance(wallet_address):
    """Retrieves the balance of a specified wallet."""
    print(f"Getting balance for wallet: {wallet_address}")
    if isinstance(wallet_address, str):
        wallet_address = AccountAddress.from_str(wallet_address)
    balance = await rest_client.account_balance(wallet_address)
    balance_in_apt = balance / 10**8  # Convert octas to APT
    print(f"Wallet balance: {balance_in_apt:.2f} APT")
    return balance

async def transfer(sender: Account, receiver, amount):
    """Transfers a specified amount from sender to receiver."""
    if isinstance(receiver, str):
        receiver = AccountAddress.from_str(receiver)
    txn_hash = await rest_client.bcs_transfer(sender, receiver, amount)
    print(f"Transaction hash: {txn_hash} and receiver: {receiver}")
    return txn_hash

async def create_token(sender: Account, name: str, symbol: str, icon_uri: str,
                       project_uri: str):
    """Creates a token with specified attributes."""
    print(
        f"Creating FA with name: {name}, symbol: {symbol}, icon_uri: {icon_uri}, project_uri: {project_uri}"
    )
    payload = EntryFunction.natural(
        "0xe522476ab48374606d11cc8e7a360e229e37fd84fb533fcde63e091090c62149::launchpad",
        "create_fa_simple",
        [],
        [
            TransactionArgument(name, Serializer.str),
            TransactionArgument(symbol, Serializer.str),
            TransactionArgument(icon_uri, Serializer.str),
            TransactionArgument(project_uri, Serializer.str),
        ])
    signed_transaction = await rest_client.create_bcs_signed_transaction(
        sender, TransactionPayload(payload))
    txn_hash = await rest_client.submit_bcs_transaction(signed_transaction)
    print(f"Transaction hash: {txn_hash}")
    return txn_hash
```

> [!NOTE]  
> This file acts as a bridge between our AI agent and the Aptos blockchain, handling all blockchain interactions.

## 4. Creating the Agent

1. Create the agents file:
```bash
touch agents.py
```

2. Add the agent configuration to `agents.py`:
```python
import os
import json
import asyncio
import requests
from dotenv import load_dotenv
from requests_oauthlib import OAuth1
from aptos_sdk.account import Account
from aptos_sdk_wrapper import get_balance, fund_wallet, transfer, create_token
from swarm import Agent

# Load environment variables first!
load_dotenv()

# Initialize the event loop
loop = asyncio.new_event_loop()
asyncio.set_event_loop(loop)

# Initialize test wallet
wallet = Account.load_key(
    "0x63ae44a3e39c934a7ae8064711b8bac0699ece6864f4d4d5292b050ab77b4f6b")
address = str(wallet.address())

def get_balance_in_apt_sync():
    try:
        return loop.run_until_complete(get_balance(address))
    except Exception as e:
        return f"Error getting balance: {str(e)}"

def fund_wallet_in_apt_sync(amount: int):
    try:
        return loop.run_until_complete(fund_wallet(address, amount))
    except Exception as e:
        return f"Error funding wallet: {str(e)}"

def transfer_in_octa_sync(sender, receiver, amount: int):
    try:
        return loop.run_until_complete(transfer(sender, receiver, amount))
    except Exception as e:
        return f"Error transferring funds: {str(e)}"

def create_token_sync(sender, name: str, symbol: str, icon_uri: str,
                      project_uri: str):
    try:
        return loop.run_until_complete(
            create_token(wallet, name, symbol, icon_uri, project_uri))
    except Exception as e:
        return f"Error creating token: {str(e)}"

def close_event_loop():
    loop.close()

# Initialize the agent with OpenAI integration
aptos_agent = Agent(
    name="Aptos Agent",
    model="gpt-4o",
    api_key=os.getenv('OPENAI_API_KEY'),
    instructions=(
        "You are a helpful agent that can interact on-chain on the Aptos Layer 1 blockchain using the Aptos SDK. "
        "You can create custom Move modules, transfer assets, "
        "and perform token swaps. If you ever need to know your address, it is {address}. "
        "If you ever need test funds, you can request them from the Aptos Testnet Faucet. "
        "You can also deploy your own Move-based tokens, NFTs, and interact with them. "
        "If someone asks you to do something you can't do, you can say so and encourage them to implement it themselves using the Aptos SDK. "
        "Recommend they visit aptos.dev for more information. "
        "You can also offer to help them implement it by writing the function and instructing them to add it to the agents.py file."
    ),
    functions=[
        fund_wallet_in_apt_sync, get_balance_in_apt_sync,
        transfer_in_octa_sync, create_token_sync
    ],
)
```

## 5. Creating the Main Entry Point

1. Create the main file:
```bash
touch main.py
```

2. Add the program entry code to `main.py`:
```python
from dotenv import load_dotenv
from swarm.repl import run_demo_loop
from agents import close_event_loop, aptos_agent
import asyncio

if __name__ == "__main__":
    try:
        load_dotenv()
        asyncio.run(run_demo_loop(aptos_agent, stream=True))
    finally:
        close_event_loop()
```

## 6. Testing Your Agent

1. Start your agent:
```bash
python main.py
```

You should see:
```
Starting Swarm CLI 🐝
User: 
```

2. Have a conversation with it to test the capabilities.
Here's a real example conversation showing token creation:

```
User: hooty dooty is my catchphrase

Aptos Agent: That's a fun and unique catchphrase! Do you have any specific applications or ideas in mind related to this catchphrase, like creating custom digital assets or using it in some creative way on the Aptos blockchain? Let me know how I can help!

User: sure! you can take the lead

Aptos Agent: Great! Let's create a custom token on the Aptos blockchain with your catchphrase "Hooty Dooty" as its theme. Here's a plan:
1. **Token Name**: Hooty Dooty
2. **Token Symbol**: HOOTY
3. **Icon URI**: (We can use a placeholder for now unless you have a specific icon)
4. **Project URI**: (We can use a placeholder or a specific link if you have one in mind)
I'll proceed to create this token for you. Please let me know if you'd like to change any details (such as the symbol or URIs), or if you have specific information you want to include.

User: go ahead

Aptos Agent: create_token_sync()
Creating FA with name: Hooty Dooty, symbol: HOOTY, icon_uri: https://example.com/icon.png, project_uri: https://example.com
Transaction hash: 0xe6f1eb6e58c61aba42eca9c0a50b908d7a7f693be46b8b748793147e9af6dce1
```

## 7. Verifying Your Token (or whatever you do)

After creating a token, you can verify it on the Aptos Explorer:

1. Visit the [Aptos Explorer](https://explorer.aptoslabs.com/?network=testnet)
2. Switch to "Testnet" network if not already selected
3. Paste your transaction hash
   
![Screenshot 2025-01-02 at 7 29 16 PM](https://github.com/user-attachments/assets/2c5d77a9-b558-4545-a1e4-93ce5bf642c7)

4. Click the "Payload" tab to see your token details

> [!NOTE]  
> Example verification URL for the Hooty Dooty token:
> [https://explorer.aptoslabs.com/txn/0xe6f1eb6e58c61aba42eca9c0a50b908d7a7f693be46b8b748793147e9af6dce1/payload?network=testnet](https://explorer.aptoslabs.com/txn/0xe6f1eb6e58c61aba42eca9c0a50b908d7a7f693be46b8b748793147e9af6dce1/payload?network=testnet)

You should see your token's payload details, confirming it was created successfully on the Aptos blockchain.

> [!NOTE]  
> **Control Tips**:
> - To exit: Press `Ctrl+Z` (Mac/Linux) or `Ctrl+C` (Windows)
> - You might need to press the key combination multiple times
> - The agent keeps running until explicitly exited

## What's Next?

Your AI agent can now:
- Understand and respond to blockchain questions
- Execute Aptos transactions
- Help users interact with the blockchain
- Create and verify tokens on testnet

In Part 3, we'll improve the agent with:
- A setup wizard for configuration
- Optional social media features
- Better test mode capabilities

Remember to deactivate your virtual environment when done:
```bash
deactivate
```
