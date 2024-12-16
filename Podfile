source 'https://github.com/CocoaPods/Specs.git'
use_frameworks!


def shared_pods
    pod 'SwiftyBeaver'
    #pod 'OpenSSL-Apple', :podspec => 'https://www.project3851.com/podspecs/OpenSSL-Apple.podspec'
    pod 'OpenSSL-Apple', :podspec => 'https://raw.githubusercontent.com/onetown/openssl-apple/refs/heads/master/OpenSSL-Apple.podspec'
    #pod 'OpenSSL-Apple' , :path => '/Users/liangbo/development/onetown/openssl-apple'
end

abstract_target 'TunnelKit' do
    target 'TunnelKit-iOS' do
        platform :ios, '12.0'
        shared_pods
    end
    target 'TunnelKitTests-iOS' do
        platform :ios, '12.0'
    end
    target 'TunnelKitHost' do
        platform :ios, '12.0'
    end

    target 'TunnelKit-macOS' do
        platform :osx, '10.15'
        shared_pods
    end
    target 'TunnelKitTests-macOS' do
        platform :osx, '10.15'
    end
end
