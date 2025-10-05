# FHE_1.1
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "fhevm/lib/TFHE.sol";

contract PrivateVoting {
    // === Variables de Estado ===

    // Llevaremos la cuenta de los votos "Sí" y "No" de forma encriptada.
    // Usamos euint32 para permitir un gran número de votos.
    euint32 private yesVotes;
    euint32 private noVotes;

    // Usamos un 'mapping' para registrar qué dirección ya ha votado.
    // Es público para que cualquiera pueda verificar SI alguien ha votado, pero no QUÉ ha votado.
    mapping(address => bool) public hasVoted;

    // === Eventos ===
    event Voted(address indexed voter);

    // === Lógica del Contrato ===

    /**
     * @dev Permite a un usuario emitir un voto encriptado (true para "Sí", false para "No").
     * @param encryptedVote El voto del usuario como un booleano encriptado (ebool).
     */
    function vote(bytes calldata encryptedVote) public {
        // 1. Verificar que el votante no haya votado antes.
        require(!hasVoted[msg.sender], "Error: Ya has votado.");

        // 2. Registrar que esta dirección ya votó para evitar el doble voto.
        hasVoted[msg.sender] = true;

        // 3. Deserializar el voto encriptado que nos envió el usuario.
        ebool choice = TFHE.asEbool(encryptedVote);

        // 4. Lógica de conteo de votos usando FHE (la parte más importante).
        // Creamos valores encriptados para 1 y 0 que usaremos para sumar.
        euint32 encryptedOne = TFHE.asEuint32(1);
        euint32 encryptedZero = TFHE.asEuint32(0);

        // Usamos TFHE.cmux (Conditional Multiplexer). Es como un if/else para datos encriptados.
        // cmux(condición, valor_si_true, valor_si_false)

        // Si 'choice' es true (voto "Sí"), suma 1 a yesVotes y 0 a noVotes.
        yesVotes = TFHE.add(yesVotes, TFHE.cmux(choice, encryptedOne, encryptedZero));

        // Si 'choice' es false (voto "No"), suma 1 a noVotes y 0 a yesVotes.
        // Hacemos esto invirtiendo la condición con TFHE.not().
        noVotes = TFHE.add(noVotes, TFHE.cmux(TFHE.not(choice), encryptedOne, encryptedZero));
        
        emit Voted(msg.sender);
    }

    /**
     * @dev Revela el resultado de la votación sin exponer los conteos exactos.
     * @return yesWins Un booleano encriptado que es true si "Sí" ganó.
     * @return noWins Un booleano encriptado que es true si "No" ganó.
     * @return isTie Un booleano encriptado que es true si fue un empate.
     */
    function getResults() public view returns (ebool yesWins, ebool noWins, ebool isTie) {
        yesWins = TFHE.gt(yesVotes, noVotes); // gt = Greater Than
        noWins = TFHE.lt(yesVotes, noVotes);  // lt = Less Than
        isTie = TFHE.eq(yesVotes, noVotes);   // eq = Equal
        return (yesWins, noWins, isTie);
    }
}
