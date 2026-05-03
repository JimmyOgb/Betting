# Betting
# v0.2.16
# { "Depends": "py-genlayer:1jb45aa8ynh2a9c9xn3b7qqh8sm5q93hwfp7jqmwsfhh8jpz09h6" }

from genlayer import *
import json
import typing

class BettingHub(gl.Contract):
    # State variables
    owner: str
    bet_description: str
    is_active: bool
    total_pot: int
    winner_declared: str
    # Maps are handled as dicts in the current VM version
    # Note: Address keys are usually strings in GenLayer Studio
    bets: dict[str, int]
    choices: dict[str, str]

    def init(self, description: str):
        self.owner = gl.msg.sender
        self.bet_description = description
        self.is_active = True
        self.total_pot = 0
        self.winner_declared = ""
        self.bets = {}
        self.choices = {}

    @gl.public.write
    @gl.payable
    def place_bet(self, selection: str):
        """
        Allows a user to place a bet. 
        'selection' is what the user is betting on (e.g., 'Team A' or 'Heads').
        """
        if not self.is_active:
            gl.revert("Betting is closed for this event")
        
        amount = int(gl.msg.value)
        if amount <= 0:
            gl.revert("Must bet a positive amount of GLT")

        user = gl.msg.sender
        
        # Track the bet
        self.bets[user] = self.bets.get(user, 0) + amount
        self.choices[user] = selection.lower().strip()
        self.total_pot += amount
        
        return f"Bet of {amount} placed on {selection}"

    @gl.public.write
    def resolve_market(self, resolution_source_url: str) -> str:
        """
        Uses LLM consensus to determine the winner from a web source.
        """
        if gl.msg.sender != self.owner:
            gl.revert("Only the creator can trigger resolution")
        
        if not self.is_active:
            gl.revert("Already resolved")

        desc = self.bet_description

        def get_consensus_winner():
            # Scrape data from the provided URL
            web_data = gl.nondet.web.render(resolution_source_url, mode="text")
            
            prompt = f"""
            Search the following text to find the winner of: {desc}
            
            Web Data:
            {web_data}
            
            Return ONLY a JSON object:
            {{
                "winner": "name_of_winner_or_outcome"
            }}
            """
            raw = gl.nondet.exec_prompt(prompt)
            # Standard cleanup for JSON parsing
            clean_json = raw.replace("```json", "").replace("```", "").strip()
            return json.loads(clean_json)

        # Strict agreement between nodes on the winner
        result = gl.eq_principle.strict_eq(get_consensus_winner)
        
        self.winner_declared = result["winner"].lower().strip()
        self.is_active = False
        
        return f"Market resolved! Winner: {self.winner_declared}"

    @gl.public.write
    def claim_winnings(self):
        """
        Simple distribution: If you chose correctly, you get your share.
        (Note: In a production app, you'd calculate proportional payouts).
        """
        if self.is_active:
            gl.revert("Market not resolved yet")
        
        user = gl.msg.sender
        if user not in self.bets:
            gl.revert("No bet found for this address")
            
        if self.choices[user] == self.winner_declared:
            # Simple version: return 2x bet (if pot allows)
            # In v0.2.16, use gl.transfer(target, amount)
            payout = self.bets[user] * 2 
            gl.transfer(user, payout)
            del self.bets[user] # Prevent double-claiming
            return "Winnings claimed!"
        
        return "Sorry, your bet did not win."

    @gl.public.view
    def get_market_status(self) -> dict:
        return {
            "description": self.bet_description,
            "pot": self.total_pot,
            "status": "Active" if self.is_active else "Resolved",
            "winner": self.winner_declared
        }
        
