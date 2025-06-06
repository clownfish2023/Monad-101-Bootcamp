# 第四章：Web3 前端 101

1. 完成课程的实操后，请思考如何监听合约事件；当有别人购买了像素格子的时候，如何及时通过监听事件更新 UI ? 请提交示例代码

完整代码：<https://github.com/qiaopengjun5162/buyearth>

## 合约代码

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract BuyEarth {
    uint256 private constant PRICE = 0.001 ether;
    address private owner;
    uint256[100] private squares;
    address[] private depositorList;
    mapping(address => uint256) public userDeposits;

    event BuySquare(uint8 idx, uint256 color);
    event OwnershipTransferred(address indexed oldOwner, address indexed newOwner);
    event ColorChanged(uint8 indexed idx, uint256 color);
    event Deposited(address indexed sender, uint256 amount);
    event Receive(address indexed sender, uint256 amount);

    modifier onlyOwner() {
        require(msg.sender == owner, "Only owner can call this function");
        _;
    }

    constructor() {
        owner = msg.sender;
    }

    function getSquares() public view returns (uint256[] memory) {
        uint256[] memory _squares = new uint256[](100);
        for (uint256 i = 0; i < 100; i++) {
            _squares[i] = squares[i];
        }
        return _squares;
    }

    function buySquare(uint8 idx, uint256 color) public payable {
        // === Checks ===
        require(idx < 100, "Invalid square number");
        require(msg.value >= PRICE, "Incorrect price");
        require(color <= 0xFFFFFF, "Invalid color");
        uint256 change = msg.value - PRICE;

        // === Effects ===
        squares[idx] = color;
        emit BuySquare(idx, color);

        // === Interactions ===
        if (change > 0) {
            (bool success1,) = msg.sender.call{value: change}("");
            require(success1, "Change return failed");
        }
        (bool success2,) = owner.call{value: PRICE}("");
        require(success2, "Owner payment failed");
    }

    function deposit() public payable {
        require(msg.value > 0, "Must send some ETH");

        // 如果是新存款用户，添加到列表
        if (userDeposits[msg.sender] == 0) {
            depositorList.push(msg.sender);
        }

        userDeposits[msg.sender] += msg.value;
        emit Deposited(msg.sender, msg.value);
    }

    function withdrawTo(address recipient) public onlyOwner {
        require(recipient != address(0), "Invalid recipient address");
        uint256 balance = address(this).balance;

        require(balance > 0, "No funds to withdraw");
        require(depositorList.length > 0, "No depositors");

        address[] memory depositors = _getAllDepositors(); // 获取所有存款用户
        for (uint256 i = 0; i < depositors.length; i++) {
            userDeposits[depositors[i]] = 0;
        }

        (bool success,) = recipient.call{value: balance}("");
        require(success, "Withdrawal failed");
    }

    function setOwner(address newOwner) public onlyOwner {
        require(newOwner != address(0), "Invalid owner address");
        emit OwnershipTransferred(owner, newOwner);
        owner = newOwner;
    }

    function getOwner() public view returns (address) {
        return owner;
    }

    function getColor(uint8 idx) public view returns (uint256) {
        return squares[idx];
    }

    function setColor(uint8 idx, uint256 color) public onlyOwner {
        require(idx < 100, "Invalid square number");
        squares[idx] = color;
        emit ColorChanged(idx, color);
    }

    function getUserDeposits(address user) public view returns (uint256) {
        return userDeposits[user];
    }

    function _getAllDepositors() private view returns (address[] memory) {
        return depositorList;
    }

    receive() external payable {
        emit Receive(msg.sender, msg.value);
        deposit();
    }
}
```
