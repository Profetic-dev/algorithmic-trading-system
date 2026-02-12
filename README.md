# Algorithmic Trading System

An autonomous trading system built on original market structure theory.

Built by [PROFETIC](https://profetic.dev)

---

## Overview

Markets aren't random — they have structure. This system maps that structure and trades it.

Starting from first principles, we developed a fractal taxonomy of market behavior, then built an autonomous system to identify and act on structural transitions.

```
Market Data → Indicators → Signal Detection → State Machine → Execution
     ↓            ↓              ↓                ↓             ↓
  Live feed    8 lenses     Convergence      Position      Exchange API
  + history    on structure   validation      management    integration
```

**Status:** Currently running in production.

---

## Architecture

### 1. Signal Detection System

Multi-indicator convergence using typed signal structures:

```python
@dataclass
class EntrySignal:
    """Entry signal with full context"""
    time: pd.Timestamp
    price: float
    indicators: List[str]      # Which indicators fired
    indicator_count: int       # Convergence count
    range_12h: float          # Market context
    momentum_24h: float       # Trend context
    support_4h: float         # Structure level

@dataclass  
class ExitSignal:
    """Exit signal with timing and momentum info"""
    time: pd.Timestamp
    signal_time: pd.Timestamp
    exit_time: pd.Timestamp
    indicators: List[str]
    divergence_count: int      # Multi-timeframe divergences
    delay_hours: int           # Momentum-based delay
    momentum_category: str     # Current regime
    exit_type: str
```

### 2. Indicator State Calculation

Each indicator is a different "lens" on market structure:

```python
class SignalDetector:
    """Detect entry and exit signals using indicator convergence"""
    
    def __init__(self):
        # Entry requires convergence across multiple dimensions
        self.entry_indicators = [
            'volume_decline',      # Accumulation signature
            'bb_lower_touch',      # Volatility extreme
            'bb_squeeze',          # Compression pattern
            'rsi_oversold',        # Momentum extreme
            'rsi_oversold_cross',  # Momentum reversal
            'stoch_oversold',      # Short-term extreme
            'macd_cross_up',       # Trend shift
            'ma_cross_up'          # Structural cross
        ]
        
        # Exit uses different indicators (divergence-based)
        self.exit_indicators = [
            'rsi_9_ob',            # Fast momentum
            'rsi_14_ob',           # Standard momentum
            'stoch_14_ob',         # Short-term extreme
            'stoch_21_ob',         # Medium-term extreme
            'bb_20_touch',         # Volatility band
            'bb_30_touch',         # Extended band
            'volume_spike',        # Distribution signature
            'macd_bearish'         # Trend exhaustion
        ]
    
    def calculate_entry_states(self, df: pd.DataFrame) -> pd.DataFrame:
        """Calculate boolean states for all entry indicators"""
        states = pd.DataFrame(index=df.index)
        
        # Each indicator calculates its own state
        # Example: Volume decline = current < 50% of 20-period MA
        states['volume_decline'] = df['volume'] < (0.5 * df['volume_ma_20'])
        states['volume_decline_price'] = df['close'].where(states['volume_decline'])
        
        # BB lower touch = price touched lower band
        states['bb_lower_touch'] = df['low'] <= df['bb_lower']
        states['bb_lower_touch_price'] = df['close'].where(states['bb_lower_touch'])
        
        # ... additional indicators ...
        
        return states
```

### 3. State Machine Engine

The core trading logic operates as a finite state machine:

```python
async def _trading_loop(self) -> None:
    """Main trading loop with state machine logic"""
    
    while self.is_running:
        # Get current portfolio state
        portfolio = self.position_tracker.get_portfolio_state()
        
        # Prevent state ping-ponging after trades
        if self.last_action == 'exit' and portfolio['state'] == 'incomplete_entry':
            portfolio['state'] = 'ready_to_enter'
            self.last_action = None
        
        # Check risk manager
        can_trade = self.risk_manager.check_can_trade()
        if not can_trade['allowed']:
            self.logger.error(f"Trading halted: {can_trade['reason']}")
            await asyncio.sleep(60)
            continue
        
        # State machine transitions
        if portfolio['state'] == 'ready_to_enter':
            await self._check_entry_signals(portfolio)
            
        elif portfolio['state'] == 'ready_to_exit':
            await self._check_exit_signals(portfolio)
            
        elif portfolio['state'] == 'incomplete_entry':
            # Handle partial fills
            if self.current_entry_signal is None:
                await self._check_entry_signals(portfolio)
            else:
                await self._complete_entry(portfolio)
                
        elif portfolio['state'] == 'incomplete_exit':
            # Validate we have sufficient divergences
            if self.divergence_count < 3:
                self.logger.info(f"Insufficient divergences ({self.divergence_count})")
            # ... handle exit completion ...
```

### 4. Position Tracking

Ground truth from exchange, not internal state:

```python
class LivePositionTracker:
    """Track real positions from exchange trade history"""
    
    def calculate_position_from_trades(self, lookback_hours: int = 600):
        """
        Calculate exact position from exchange trade history.
        This is GROUND TRUTH - not internal tracking.
        """
        # Get trade history from exchange
        result, success = self.client.get_trades_history(start=start_time)
        
        trades_list = []
        for trade_id, trade in trades_data.items():
            trades_list.append({
                'trade_id': trade_id,
                'time': trade['time'],
                'type': trade['type'],     # 'buy' or 'sell'
                'volume': float(trade['vol']),
                'price': float(trade['price']),
                'cost': float(trade['cost']),
                'fee': float(trade['fee']),
            })
        
        # Calculate running position
        running_balance = 0.0
        for trade in sorted(trades_list, key=lambda x: x['time']):
            if trade['type'] == 'buy':
                running_balance += trade['volume']
            else:
                running_balance -= trade['volume']
        
        return running_balance
```

### 5. Risk Management

Minimal — only protects against technical failures:

```python
class MinimalRiskManager:
    """
    Minimal risk manager - technical protection only.
    
    Philosophy: The strategy handles market risk.
    This only catches impossible data and API failures.
    """
    
    def __init__(self):
        # Technical limits only
        self.max_consecutive_api_errors = 50
        self.min_price = 0.10              # Below this = bad data
        self.max_spike_pct = 100.0         # 100% in 1 min = impossible
        
        # NO trading limits - strategy decides that
        # NO max losses - strategy handles exits
        # NO position limits - strategy manages size
    
    def check_price_sanity(self, price: float) -> Dict[str, Any]:
        """
        Check if price data is sane.
        Only catches impossible values, not market moves.
        """
        if price < self.min_price:
            return {'valid': False, 'reason': 'Price below minimum'}
        
        if self.last_valid_price:
            change_pct = abs((price - self.last_valid_price) / self.last_valid_price) * 100
            if change_pct > self.max_spike_pct:
                return {'valid': False, 'reason': f'Impossible spike: {change_pct}%'}
        
        self.last_valid_price = price
        return {'valid': True}
    
    def check_can_trade(self) -> Dict[str, Any]:
        """Only halt for technical failures, never for market conditions"""
        if self.technical_halt:
            return {'allowed': False, 'reason': self.halt_reason}
        return {'allowed': True}
```

### 6. Self-Healing Design

The system recovers from errors without manual intervention:

```python
# State persistence for crash recovery
def save_state(self):
    """Save state to disk for crash recovery"""
    state = {
        'last_timestamp': self.last_timestamp,
        'consecutive_errors': self.consecutive_errors,
        'last_update': datetime.now().isoformat()
    }
    with open(self.state_file, 'w') as f:
        json.dump(state, f)

# Position reconstruction from exchange
# Even if local state is lost, we rebuild from ground truth
position = self.position_tracker.calculate_position_from_trades()

# Graceful degradation
if self.consecutive_errors > threshold:
    self.logger.warning("Elevated errors - continuing with caution")
    # Don't halt, just log
```

---

## Design Principles

### Convergence Over Prediction

No single indicator triggers action. Entry requires **7/8 indicators** firing within a time window. This eliminates most false signals at the cost of some missed opportunities.

### Ground Truth From Exchange

Internal state can drift. The system always validates against actual exchange data:
- Position size from trade history
- Balances from account query
- Order status from order book

### Technical Risk Only

The strategy handles market risk through entry/exit logic. The risk manager only catches:
- Bad data (impossible prices)
- API failures (connection issues)
- Technical halts (system problems)

It never overrides the strategy for market reasons.

---

## Tech Stack

- **Language:** Python 3.11+
- **Performance:** NumPy, pandas, Numba JIT
- **Exchange:** Kraken API (REST + WebSocket)
- **Data:** 1-minute candles, 24h+ history

---

## Private Repository

This is a showcase repository demonstrating architecture and design patterns.

The trading system is proprietary and not available for licensing.

**Contact:** [hello@profetic.dev](mailto:hello@profetic.dev)

---

© 2025 PROFETIC
