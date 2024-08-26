# 质押量

* `普通用户的质押可随进随出，用户提取 (unbond) 质押代币后，收益(分到的出块 + 气费收益 - 佣金)立刻到账，质押的 token 会在经过 params.bondingtime (21 天) 后到账。`
* `起始验证节点自己最小质押 30 万枚，(--min-self-delegation=300000sofa), 起始验证节点可以解除质押，但是`
  * &#x20;`如果解除质押后剩余的 token < 30 万: unbonding 交易会失败`
  * `如果解除质押后剩余 token >= 30 万: 收益立刻到账，解除质押的 token 会在 21 天后到账`
* 实现
  *   在区块高度 1 的 end block hook 里获取 block 1 的 block time 和 validator 列表

      <pre class="language-go"><code class="lang-go">GenesisTime = sdkCtx.BlockTime()
      for _, validator := range validators {    
           GenesisValidators = append(GenesisValidators, validator.OperatorAddress)
      <strong>}
      </strong></code></pre>
*   undelegation.go 的 Undelegate 里面，增加逻辑，判断 validator 是否在上述的 validator 列表中，并判断剩下的质押 token 是否大于 `min-self-delegation`

    ```go
    isValidatorOperator := delAddr.Equals(valAddr)
     
    // set the unbonding completion time and completion height appropriately
    if delAddr != nil && valAddr != nil && delAddr.Equals(valAddr) {
        // self unbond
        if isValidatorOperator && validator.TokensFromShares(validator.DelegatorShares).TruncateInt().Sub(sharesAmount.TruncateInt()).LT(validator.MinSelfDelegation) && contains(GenesisValidators, validator.OperatorAddress) {
            validatorUnbondingTime, err := k.GenesisValidatorUnbondingTime(ctx)
            if err != nil {
                return time.Time{}, math.Int{}, err
            }
            if sdkCtx.BlockTime().Before(GenesisTime.Add(validatorUnbondingTime)) {
                return time.Time{}, math.Int{}, errors.New("the initial validator needs to be staked for at least 3 years")
            }  
        }
    }
    ```

\
