//
// DiffPrivAnalytics.swift
// Minimal differential-privacy aggregator (Laplace mechanism demo)
// Swift 5+
//

import Foundation
import GameplayKit // optional for RNG if needed

// Laplace sampler
func laplaceSample(scale: Double) -> Double {
    let u = Double.random(in: -0.5..<0.5)
    return -scale * sign(of: u) * log(1 - 2 * abs(u))
}
func sign(of x: Double) -> Double { x >= 0 ? 1 : -1 }

struct PrivacyAccountant {
    let totalEps: Double
    private(set) var used: Double = 0
    mutating func charge(eps: Double) -> Bool {
        if used + eps > totalEps { return false }
        used += eps
        return true
    }
    func remaining() -> Double { totalEps - used }
}

struct DPDB {
    private var data: [[Double]] = []
    var sensitivity: Double = 1.0
    var accountant: PrivacyAccountant
    
    init(epsBudget: Double) {
        accountant = PrivacyAccountant(totalEps: epsBudget)
    }
    
    mutating func insert(row: [Double]) {
        data.append(row)
    }
    
    mutating func dpSum(col: Int, eps: Double) -> Double? {
        guard accountant.charge(eps: eps) else { return nil }
        let s = data.reduce(0.0) { $0 + ($1.indices.contains(col) ? $1[col] : 0.0) }
        let scale = sensitivity / eps
        let noise = laplaceSample(scale: scale)
        return s + noise
    }
    
    mutating func dpMean(col: Int, eps: Double) -> Double? {
        // split budget
        let epsSum = eps * 0.7
        let epsCount = eps * 0.3
        guard accountant.charge(eps: epsSum) else { return nil }
        guard accountant.charge(eps: epsCount) else { return nil }
        let s = data.reduce(0.0) { $0 + ($1.indices.contains(col) ? $1[col] : 0.0) }
        let count = Double(data.count)
        let noisySum = s + laplaceSample(scale: sensitivity/epsSum)
        let noisyCount = count + laplaceSample(scale: 1.0/epsCount)
        return noisySum / max(1.0, noisyCount)
    }
}

// Demo
var db = DPDB(epsBudget: 1.0)
for _ in 0..<100 {
    let row = [Double.random(in: 0...100), Double.random(in: 0...1)]
    db.insert(row: row)
}

if let s = db.dpSum(col: 0, eps: 0.4) {
    print("DP Sum (eps=0.4):", s)
} else {
    print("Budget exceeded for dpSum")
}

if let m = db.dpMean(col: 0, eps: 0.4) {
    print("DP Mean (eps=0.4):", m)
} else {
    print("Budget exceeded for dpMean")
}

print("Remaining budget:", db.accountant.remaining())
